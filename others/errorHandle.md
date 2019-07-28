## React中的错误处理

React作为一个UI库有自己的错误搜集和处理机制，在React中`ReactErrorUtils`是处理错误相关的一个文件。

#### 用于否发生错误的状态和捕获到的错误
```
// Used by Fiber to simulate a try-catch.
let hasError: boolean = false;
let caughtError: mixed = null;
```
#### 记录是否重新抛出错误和重新抛出的错误
```
// Used by event system to capture/rethrow the first error.
let hasRethrowError: boolean = false;
let rethrowError: mixed = null;
```

#### reporter对象上
该对象的onError方法，方法体中记录错误的状态和捕获到的错误
```
const reporter = {
  onError(error: mixed) {
    hasError = true;
    caughtError = error;
  },
};
```

在React源码中经常会见到`invokeGuardedCallback`这种函数，一开始没怎么看懂，后来才知道是处理错误的方法。


#### invokeGuardedCallback

从其名字上可以看到是调用一个被保护的callback,意思是调用一个函数，同时防止其中发生的错误。

参数：

1. name 用于日志或者调试
2. 要被执行的函数
3. 函数调用的context
4. 其他参数



```
export function invokeGuardedCallback<A, B, C, D, E, F, Context>(
  name: string | null,
  func: (a: A, b: B, c: C, d: D, e: E, f: F) => mixed,
  context: Context,
  a: A,
  b: B,
  c: C,
  d: D,
  e: E,
  f: F,
): void {
  hasError = false;
  caughtError = null;
  //通过apply将this指向reporter，当invokeGuardedCallbackImpl发生错误的时候， 调用reporter的onError 方法将错误存储到reporter的最外层的hasError和caughtError 变量上。

  invokeGuardedCallbackImpl.apply(reporter, arguments);
}
```

#### invokeGuardedCallbackAndCatchFirstError

看起来和invokeGuardedCallback一样，只是对invokeGuardedCallback的封装，当执行完invokeGuardedCallback之后，通过hasError判断是否发生了错误，并调用clearCaughtError返回错误对象，并清空hasError、caughtError。然后将返回的错误存储到rethrowError，并改变hasRethrowError的状态。

```
export function invokeGuardedCallbackAndCatchFirstError<
  A,
  B,
  C,
  D,
  E,
  F,
  Context,
>(
  name: string | null,
  func: (a: A, b: B, c: C, d: D, e: E, f: F) => void,
  context: Context,
  a: A,
  b: B,
  c: C,
  d: D,
  e: E,
  f: F,
): void {
  invokeGuardedCallback.apply(this, arguments);
  if (hasError) {
    const error = clearCaughtError();
    if (!hasRethrowError) {
      hasRethrowError = true;
      rethrowError = error;
    }
  }
}

export function rethrowCaughtError() {
  if (hasRethrowError) {
    const error = rethrowError;
    hasRethrowError = false;
    rethrowError = null;
    throw error;
  }
}

export function hasCaughtError() {
  return hasError;
}

export function clearCaughtError() {
  if (hasError) {
    const error = caughtError;
    hasError = false;
    caughtError = null;
    return error;
  } else {
    invariant(
      false,
      'clearCaughtError was called but no error was captured. This error ' +
        'is likely caused by a bug in React. Please file an issue.',
    );
  }
}
```

#### invokeGuardedCallbackImpl

invokeGuardedCallbackImpl在不同环境实现不同

##### 生产环境

```
let invokeGuardedCallbackImpl = function<A, B, C, D, E, F, Context>(
  name: string | null,
  func: (a: A, b: B, c: C, d: D, e: E, f: F) => mixed,
  context: Context,
  a: A,
  b: B,
  c: C,
  d: D,
  e: E,
  f: F,
) {
  const funcArgs = Array.prototype.slice.call(arguments, 3);
  try {
    func.apply(context, funcArgs);
  } catch (error) {
      //this指向reporter
    this.onError(error);
  }
};
```
可以看到生产环境使用`try...catch`捕获错误

##### 开发环境


在开发环境为了更好的配合浏览器的DevTools，主要原因是为了防止浏览器 DevTool 的pause on any exception功能导致 React 渲染因为出现错误被中断，而导致渲染不出内容，因为 React 现在是可以捕获错误并且在错误的时候渲染对应的 UI 的，但是浏览器的这个功能甚至可以让被try catch捕获的错误也停在错误出现的地方，对 React 开发的体验不是很好。

所以React在开发环境下利用自定义事件模拟了try ... catch功能

具体方法是：定义一个fake node ，然后同步的dispatch一个fake event,在事件处理的hanlder中调用传入的回调函数，如果回调函数出现错误，错误就会被全局的事件handler捕获。


```
if (__DEV__) {
  if (
    typeof window !== 'undefined' &&
    typeof window.dispatchEvent === 'function' &&
    typeof document !== 'undefined' &&
    typeof document.createEvent === 'function'
  ) {
    const fakeNode = document.createElement('react');

    const invokeGuardedCallbackDev = function<A, B, C, D, E, F, Context>(
      name: string | null,
      func: (a: A, b: B, c: C, d: D, e: E, f: F) => mixed,
      context: Context,
      a: A,
      b: B,
      c: C,
      d: D,
      e: E,
      f: F,
    ) {
      //创建一个fake event
      const evt = document.createEvent('Event');

      This strategy works even if the browser is flaky and
      fails to call our global error handler, because it doesn't rely on
      the error event at all.
      // 记录传入的需要调用的函数是否有错误,初始设为true,在函数调用完之后设置为false，如果函数发生了错误，就不会被设置为false。

      //这中策略是因为即使浏览器不稳定，无法调用我们的全局错误处理程序，依然能捕获到错误，因为不依赖与error event.
      let didError = true;

      let windowEvent = window.event;

      
      const windowEventDescriptor = Object.getOwnPropertyDescriptor(
        window,
        'event',
      );
        //给 fake event创建一个事件handler，用`dispatchEvent`同步的dispatch 一个事件。在event handler 中调用传入的回调函数

        // 获取需要传递给回调的参数
      const funcArgs = Array.prototype.slice.call(arguments, 3);
      function callCallback() {
          //事件handler调用时就remove fakeNode的监听器，以保证嵌套的`invokeGuardedCallback`调用不会冲突。否则会调用stack中顶层的event handler 
        fakeNode.removeEventListener(evtType, callCallback, false);
        if (
          typeof window.event !== 'undefined' &&
          window.hasOwnProperty('event')
        ) {
          window.event = windowEvent;
        }
        //调用传入的回调
        func.apply(context, funcArgs);
        //将didError置为false
        didError = false;
      }

    //创建一个全局的错误事件handler，用来捕获错误
      
      
      let didSetError = false;
      let isCrossOriginError = false;

      function handleWindowError(event) {
        error = event.error;
        didSetError = true;
        if (error === null && event.colno === 0 && event.lineno === 0) {
            // 当加载自不同域的脚本中发生语法错误时，为避免信息泄露，语法错误的细节将不会报告，而代之简单的"Script error."
            //  event：
            // message：错误信息（字符串）。可用于HTML onerror=""处理程序中的event。
            // source：发生错误的脚本URL（字符串）
            // lineno：发生错误的行号（数字）
            // colno：发生错误的列号（数字）
            // error：Error对象（对象）
          isCrossOriginError = true;
        }
        //有的error handler被默认阻止了，先将错误记录下来
        if (event.defaultPrevented) {
            
          if (error != null && typeof error === 'object') {
            try {
              error._suppressLogging = true;
            } catch (inner) {
              // Ignore.
            }
          }
        }
      }

      // 创建一个 fake event type.
      const evtType = `react-${name ? name : 'invokeguardedcallback'}`;

      // 绑定event handlers
      window.addEventListener('error', handleWindowError);
      fakeNode.addEventListener(evtType, callCallback, false);

      //dispatch 事件
      //initEvent的第一个false表示阻止该事件向上冒泡，第二个false表示该事件默认动作不可取消
      evt.initEvent(evtType, false, false);
      fakeNode.dispatchEvent(evt);

    //还原window.event
      if (windowEventDescriptor) {
        Object.defineProperty(window, 'event', windowEventDescriptor);
      } 
    //传入的回调函数发生了错误后，didError不会被设置为false
      if (didError) {
        if (!didSetError) {
            //传入的回调函数发生了错误，但是handleWindowError没有被触发。
         
          error = new Error(
            'An error was thrown inside one of your components, but React ' +
              "doesn't know what it was. This is likely due to browser " +
              'flakiness. React does its best to preserve the "Pause on ' +
              'exceptions" behavior of the DevTools, which requires some ' +
              "DEV-mode only tricks. It's possible that these don't work in " +
              'your browser. Try triggering the error in production mode, ' +
              'or switching to a modern browser. If you suspect that this is ' +
              'actually an issue with React, please file an issue.',
          );
        } else if (isCrossOriginError) {
          error = new Error(
            "A cross-origin error was thrown. React doesn't have access to " +
              'the actual error object in development. ' +
              'See https://fb.me/react-crossorigin-error for more information.',
          );
        }
        //调用reporter记录错误
        this.onError(error);
      }

      // 移除 event listeners
      window.removeEventListener('error', handleWindowError);
    };

    invokeGuardedCallbackImpl = invokeGuardedCallbackDev;
  }
}

```