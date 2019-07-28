## SyntheticEvent

React有很多种合成事件，它们直接或者间接继承SyntheticEvent的。

如
```
SyntheticCompositionEvent
SyntheticInputEvent
SyntheticUIEvent
SyntheticMouseEvent
SyntheticPointerEvent
SyntheticAnimationEvent
SyntheticClipboardEvent
SyntheticFocusEvent
SyntheticKeyboardEvent
SyntheticDragEvent
SyntheticTouchEvent
SyntheticTransitionEvent
SyntheticWheelEvent
```

SyntheticEvent 的会构造合成事件对象 即events

```
function SyntheticEvent(
  dispatchConfig,
  targetInst,
  nativeEvent,
  nativeEventTarget,
){...}
```

参数含义：

1. dispatchConfig 含有dependencies，phasedRegistrationNames等属性的对象
2. targetInst 事件目标对应fiber node
3. nativeEvent 原生事件
4. nativeEventTarget 原生事件目标，即dom node


```
function SyntheticEvent(
  dispatchConfig,
  targetInst,
  nativeEvent,
  nativeEventTarget,
) {
    //将事件的新属性赋值给实例 
  this.dispatchConfig = dispatchConfig;
  this._targetInst = targetInst;
  this.nativeEvent = nativeEvent;
//遍历接口对象，将原生事件的属性赋值给实例,设置target属性
  const Interface = this.constructor.Interface;
  for (const propName in Interface) {
    if (!Interface.hasOwnProperty(propName)) {
      continue;
    }
    const normalize = Interface[propName];
    if (normalize) {
      this[propName] = normalize(nativeEvent);
    } else {
      if (propName === 'target') {
        this.target = nativeEventTarget;
      } else {
        this[propName] = nativeEvent[propName];
      }
    }
  }
    //设置isDefaultPrevented和isPropagationStopped属性
  const defaultPrevented =
    nativeEvent.defaultPrevented != null
      ? nativeEvent.defaultPrevented
      : nativeEvent.returnValue === false;
  if (defaultPrevented) {
      //默认返回true的函数
    this.isDefaultPrevented = functionThatReturnsTrue;
  } else {
      //默认返回false的函数
    this.isDefaultPrevented = functionThatReturnsFalse;
  }
  this.isPropagationStopped = functionThatReturnsFalse;
  return this;
}

SyntheticEvent.Interface = EventInterface;

const EventInterface = {
  type: null,
  target: null,
  // currentTarget is set when dispatching; no use in copying it here
  currentTarget: function() {
    return null;
  },
  eventPhase: null,
  bubbles: null,
  cancelable: null,
  timeStamp: function(event) {
    return event.timeStamp || Date.now();
  },
  defaultPrevented: null,
  isTrusted: null,
};
```

EventInterface保证SyntheicEvent对象都有该接口的属性，而且该接口是可以扩展的，不同子类可以实现不同的接口


```
SyntheticEvent.extend = function (Interface) {
  var Super = this;
// 经典的寄生组合式继承
  var E = function () {};
  E.prototype = Super.prototype;
  var prototype = new E();

  function Class() {
    return Super.apply(this, arguments);
  }
  _assign(prototype, Class.prototype);
  Class.prototype = prototype;
  Class.prototype.constructor = Class;
//扩展接口属性

  Class.Interface = _assign({}, Super.Interface, Interface);
  Class.extend = Super.extend;
  
  addEventPoolingTo(Class);

  return Class;
};

//初始化事件池，绑定getPooled和release方法到构造器
function addEventPoolingTo(EventConstructor) {
  EventConstructor.eventPool = [];
  EventConstructor.getPooled = getPooledEvent;
  EventConstructor.release = releasePooledEvent;
}
```

关于事件接口扩展，因为不同类型的事件有一些属性是有差异的，EventInterface只是提供了一些共同的或者默认的属性。

```
var SyntheticUIEvent = SyntheticEvent.extend({
  view: null,
  detail: null
});

const SyntheticMouseEvent = SyntheticUIEvent.extend({
  screenX: null,
  screenY: null,
  clientX: null,
  clientY: null,
  pageX: null,
  pageY: null,
  ...
})
```

#### SyntheticEvent的原型

```
Object.assign(SyntheticEvent.prototype, {
  preventDefault: function() {
      //拿到浏览器的原生事件，存在preventDefault则调用preventDefault方法，否则判断有无returnValue属性（兼容IE)
    this.defaultPrevented = true;
    const event = this.nativeEvent;
    if (!event) {
      return;
    }

    if (event.preventDefault) {
      event.preventDefault();
    } else if (typeof event.returnValue !== 'unknown') {
      event.returnValue = false;
    }
    this.isDefaultPrevented = functionThatReturnsTrue;
  },

  stopPropagation: function() {
    const event = this.nativeEvent;
    if (!event) {
      return;
    }

    if (event.stopPropagation) {
      event.stopPropagation();
    } else if (typeof event.cancelBubble !== 'unknown') {
      event.cancelBubble = true;
    }
    //对于冒泡事件来说，当事件触发，由子元素往父元素逐级向上遍历，会按顺序执行每层元素对应的事件回调，但如果发现当前元素对应的合成事件上的 isPropagationStopped为 true值，则遍历的循环将中断，也就是不再继续往上遍历，当前元素的所有父元素的合成事件就不会被触发，最终的效果，就和浏览器原生事件调用 e.stopPropagation()的效果是一样的。
    this.isPropagationStopped = functionThatReturnsTrue;
  },

  
   //由于在每次时间循环结束后会释放所有的已执行过的SyntheticEvent，将他们添加到事件池中，在事件回调中调用event.persist()方法可以允许你继续持有该事件对象而不被释放
  persist: function() {
    this.isPersistent = functionThatReturnsTrue;
  },

   //检查是否应将此事件释放回池中
  isPersistent: functionThatReturnsFalse,

  
   //事件释放的时候清空其属性，减少内存占用
  destructor: function() {
    const Interface = this.constructor.Interface;
    for (const propName in Interface) {
        this[propName] = null;

    }
    this.dispatchConfig = null;
    this._targetInst = null;
    this.nativeEvent = null;
    this.isDefaultPrevented = functionThatReturnsFalse;
    this.isPropagationStopped = functionThatReturnsFalse;
    this._dispatchListeners = null;
    this._dispatchInstances = null;
  },
});
```


##### getPooledEvent和releasePooledEvent

addEventPoolingTo中给EventConstructor添加了这两个方法


EventConstructor.getPooled = getPooledEvent;
EventConstructor.release = releasePooledEvent;

```
function getPooledEvent(dispatchConfig, targetInst, nativeEvent, nativeInst) {
  const EventConstructor = this;
  if (EventConstructor.eventPool.length) {
    const instance = EventConstructor.eventPool.pop();
    EventConstructor.call(
      instance,
      dispatchConfig,
      targetInst,
      nativeEvent,
      nativeInst,
    );
    return instance;
  }
  return new EventConstructor(
    dispatchConfig,
    targetInst,
    nativeEvent,
    nativeInst,
  );
}
function releasePooledEvent(event) {
  const EventConstructor = this;
  event.destructor();
  if (EventConstructor.eventPool.length < EVENT_POOL_SIZE) {
    EventConstructor.eventPool.push(event);
  }
}
```

`getPooledEvent`方法在首次触发事件的时候 EventConstructor.eventPool.length为 0，因为这个时候是第一次事件触发，对象池中没有对应的合成事件引用，所以需要初始化，后续再触发事件的时候，就无需 new了,直接从对象池中取，通过 EventConstructor.eventPool.pop()获取合成对象实例.

`releasePooledEvent`方法主要做了两件事，首先释放掉 event上属性占用的内存，然后把清理后的 event对象再放入对象池中，可以被后续事件对象二次利用。

