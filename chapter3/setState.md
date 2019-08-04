## setState()

```
setState(updater[, callback])
```

- setState()将组件状态的更改保存在队列中，并告诉React需要使用更新后的状态重新渲染此组件和其子组件，这是响应事件和处理网络请求响应的首选方法。
- setState()只是一个请求，而不是一个立即执行的指令去更新组件，React不保证state会立即更新，为了更好的性能，React会延迟执行，将多个更新请求合并在一次更新中。
- setState()并不总是立刻更新组件，它会使用批处理或延迟更新，在你调用setState()后立刻读取 this.state的值不会生效，相反，使用componentDidUpdate 或者 setState 回调函数 的任意一种方式都能读取到正确的值
- setState总是会触发重新渲染，除非shouldComponentUpdate() 返回 false ，shouldComponentUpdate()不应该包含可变对象作为条件渲染的逻辑，我们仅仅在state发生变化去调用setSate从而避免不必要的重新渲染


##### updater

```
(prevState, props) => stateChange
```
prevState 是上一个状态的引用. 它不应该直接被改变。这种变化应该是基于 prevState和 props构建的一个新对象。
例如

```
this.setState((prevState, props) => {
  return {counter: prevState.counter + props.step};
});
```
updater function 保证接受到的prevState和props是最新的
- 在批量更新时，prevState是上一个setState的updater执行结果
- 在非批量更新时，prevState是上次render后的结果

##### callback

第二个是可选的回调函数，它会在setState完成后并且组件重新渲染后立刻执行。一般来说，我们推荐使用componentDidUpdate() 来替换这个逻辑。

##### 传递一个对象作为setState的第一参数

```
this.setState({quantity: 2})
```
在同一周期中多次调用setState会被批量处理，它会执行浅合并来生成一个新的state，在同一周期靠后的setState()将会覆盖前一个setSate()的值(相同属性名)。

例如
```
this.state={
  quantity:0
}
this.setState({quantity: this.state.quantity+1});
this.setState({quantity: this.state.quantity+1});
this.setState({quantity: this.state.quantity+1});
this.setState({quantity: this.state.quantity+1});
```

由于进入批量处理，当前组件的state并没有被更新（其更新时在render阶段）,this.state始终是0,最终浅合并如下,最终结果是1。
```
Object.assign(
  {quantity:0},
  {quantity: this.state.quantity + 1},
  {quantity: this.state.quantity + 1},
  ...
)
```
> 1. 无论是否批量，setState总是按照我们写的顺序进行调用的。
> 2. 目前React 16之前的版本，只有事件处理函数中的setState会被批量更新

#### 为什么setState不立即更新
假如我们点击click事件后，parent 和 child组件都会setState,但是我们不想child 重复渲染两次。所以从优化性能角度进行了批量更新的处理。

#### 为什么在setState中this.state不是被同步更新的

即使state是同步更新的，但是props并不是，因为直到parent组件重复渲染之后我们才能知道props。

如果this.state被同步的更新，将会导致内部的状态不一致

假如以下代码可以正常工作：

```
console.log(this.state.value) // 0
this.setState({ value: this.state.value + 1 });
console.log(this.state.value) // 1
this.setState({ value: this.state.value + 1 });
console.log(this.state.value) // 2
```
需要注意的是在批量更新阶段，因为只进行一次render,如果state被同步更新的话，props并不会被更新，this.props.value仍然是旧值，导致内部状态失去了一致性。

要是props也能同步的更新，只能放弃批量更新了，这样对于性能又是一大损失。


### setState方法相关调用函数
 ##### classComponentUpdater
我们知道在React中定义的Component有第三个参数
``` Component(props, context, updater) ```
updater是在组件实例化时候注入进来的。

```
function adoptClassInstance(workInProgress, instance) {
  instance.updater = classComponentUpdater;
  workInProgress.stateNode = instance;
  // The instance needs access to the fiber so that it can schedule updates
  set(instance, workInProgress);
  {
    instance._reactInternalInstance = fakeInternalInstance;
  }
}
```

在dom环境中注入的updater就是classComponentUpdater，对于所有组件都是一样的。




```
var classComponentUpdater = {
  isMounted: isMounted,
  enqueueSetState: function (inst, payload, callback) {
    var fiber = get(inst);
    var currentTime = requestCurrentTime();
    var expirationTime = computeExpirationForFiber(currentTime, fiber);

    var update = createUpdate(expirationTime);
    update.payload = payload;
    if (callback !== undefined && callback !== null) {
      {
        warnOnInvalidCallback$1(callback, 'setState');
      }
      update.callback = callback;
    }

    flushPassiveEffects();
    enqueueUpdate(fiber, update);
    scheduleWork(fiber, expirationTime);
  },
  enqueueReplaceState: function (inst, payload, callback) {
    var fiber = get(inst);
    var currentTime = requestCurrentTime();
    var expirationTime = computeExpirationForFiber(currentTime, fiber);

    var update = createUpdate(expirationTime);
    update.tag = ReplaceState;
    update.payload = payload;

    if (callback !== undefined && callback !== null) {
      {
        warnOnInvalidCallback$1(callback, 'replaceState');
      }
      update.callback = callback;
    }

    flushPassiveEffects();
    enqueueUpdate(fiber, update);
    scheduleWork(fiber, expirationTime);
  },
  enqueueForceUpdate: function (inst, callback) {
    var fiber = get(inst);
    var currentTime = requestCurrentTime();
    var expirationTime = computeExpirationForFiber(currentTime, fiber);

    var update = createUpdate(expirationTime);
    update.tag = ForceUpdate;

    if (callback !== undefined && callback !== null) {
      {
        warnOnInvalidCallback$1(callback, 'forceUpdate');
      }
      update.callback = callback;
    }

    flushPassiveEffects();
    enqueueUpdate(fiber, update);
    scheduleWork(fiber, expirationTime);
  }
};
```


classComponentUpdater 有三个很重要的方法

- enqueueSetState
- enqueueForceUpdate
- enqueueReplaceState

大部分情况下使用的是enqueueSetState这个方法，因为setState方法会调用enqueueSetState。

enqueueSetState方法的功能如下

1. 拿到Component的Fiber实例 ，它其实等于const fiber = inst._reactInternalFiber; 
2. 计算当前fiber的过期时间
3. 创建update,更新其payload等属性
4. 调用enqueueUpdate和scheduleWork
  



接下来看看 ``` enqueueUpdate ``` 和 ``` scheduleWork ``` 方法


### enqueueUpdate


```
export function enqueueUpdate<State>(fiber: Fiber, update: Update<State>) {
  // Update queues 是懒创建的，即组件没有update是不会创建的.
  // 使用alternate属性双向连接一个当前fiber和其work-in-progress，当前fiber实例的alternate属性指向其work-in-progress，work-in-progress的alternate属性指向当前稳定fiber。
  const alternate = fiber.alternate;
  let queue1;//指向current的updateQueue
  let queue2; //指向alternate 的updateQueue
  if (alternate === null) {
    //alternate为null，对于classComponent第一次调用setState时alternate为null
    queue1 = fiber.updateQueue;
    queue2 = null;
    if (queue1 === null) {
      //首次调用setState创建updateQueue,这时的memoizedState为初始值
      queue1 = fiber.updateQueue = createUpdateQueue(fiber.memoizedState);
    }
  } else {
    // There are two owners.
    // 如果fiber树以及workinprogress树都存在，下面的逻辑则会同步两个树的update队列
    queue1 = fiber.updateQueue;//current Fiber updateQueue 
    queue2 = alternate.updateQueue;
    // 当两个树的队列至少有一个不存在的时候执行队列创建或者复制操作
    if (queue1 === null) {
      if (queue2 === null) {
        // Neither fiber has an update queue. Create new ones.
        //  两个队列都没有则根据各自的memoizedState创建update队列
        queue1 = fiber.updateQueue = createUpdateQueue(fiber.memoizedState);
        queue2 = alternate.updateQueue = createUpdateQueue(
          alternate.memoizedState,
        );
      } else {
        // 如果有一个没有则复制另一个队列给它
        // Only one fiber has an update queue. Clone to create a new one.
        queue1 = fiber.updateQueue = cloneUpdateQueue(queue2);
      }
    } else {
      if (queue2 === null) {
        // 如果有一个没有则复制另一个队列给它
        // Only one fiber has an update queue. Clone to create a new one.
        queue2 = alternate.updateQueue = cloneUpdateQueue(queue1);
      } else {
        // Both owners have an update queue.
      }
    }
  }

  if (queue2 === null || queue1 === queue2) {
    // There's only a single queue.
    // 如果只有一个树，或者两棵树队列是同一个，则将传入的更新对象添加到第一个队列中, 一般发生在首次 setState
    appendUpdateToQueue(queue1, update);
  } else {
    // 如果存在两个queues,我们需要追加这个 update到这个两个 queues.
    // 然而对于这种持久性结构的列表(updateQueue)需要保证一次update不能添加多次
    if (queue1.lastUpdate === null || queue2.lastUpdate === null) {
      // One of the queues is not empty. We must add the update to both queues.
      //  有一个队列不为空，将update添加到两个队列
      appendUpdateToQueue(queue1, update);
      appendUpdateToQueue(queue2, update);
    } else {
      // Both queues are non-empty. The last update is the same in both lists,
      // because of structural sharing. So, only append to one of the lists.
      appendUpdateToQueue(queue1, update);
      // But we still need to update the `lastUpdate` pointer of queue2.
      queue2.lastUpdate = update;
    }
  }
}

```

该方法最终会更新fiber的updateQueue,用于在recondilation的render阶段对state进行处理,在updateClassInstance方法中会processUpdateQueue。

```
var updateQueue = workInProgress.updateQueue;
if (updateQueue !== null) {
  processUpdateQueue(workInProgress, updateQueue, newProps, instance, renderExpirationTime);
  newState = workInProgress.memoizedState;
}
```

##### processUpdateQueue

```
function processUpdateQueue(workInProgress, queue, props, instance, renderExpirationTime) {
  hasForceUpdate = false;

  queue = ensureWorkInProgressQueueIsAClone(workInProgress, queue);

  {
    currentlyProcessingQueue = queue;
  }

  // These values may change as we process the queue.
  var newBaseState = queue.baseState;
  var newFirstUpdate = null;
  var newExpirationTime = NoWork;

  // 遍历updateQueue计算最终的state
  var update = queue.firstUpdate;
  var resultState = newBaseState;//之前的state,queue.baseState
  while (update !== null) {
    var updateExpirationTime = update.expirationTime;
    if (updateExpirationTime < renderExpirationTime) {
      // 优先级不够就跳过 ，通过newFirstUpdate保留第一个被跳过的update
      if (newFirstUpdate === null) {
        // This is the first skipped update. It will be the first update in
        // the new list.
        newFirstUpdate = update;
        // Since this is the first update that was skipped, the current result
        // is the new base state.
        newBaseState = resultState;
      }
      // 由于跳过的update会保留在列表中，更新newExpirationTime
      
      if (newExpirationTime < updateExpirationTime) {
        newExpirationTime = updateExpirationTime;
      }
    } else {
      // This update does have sufficient priority. Process it and compute
      // a new result.
      //处理更新，计算state,将结果赋值resultState
      resultState = getStateFromUpdate(workInProgress, queue, update, resultState, props, instance);
      var _callback = update.callback;
      if (_callback !== null) {
        workInProgress.effectTag |= Callback;
        // Set this to null, in case it was mutated during an aborted render.
        update.nextEffect = null;
        if (queue.lastEffect === null) {
          queue.firstEffect = queue.lastEffect = update;
        } else {
          queue.lastEffect.nextEffect = update;
          queue.lastEffect = update;
        }
      }
    }
    // Continue to the next update.
    update = update.next;
  }

  // Separately, iterate though the list of captured updates.
  var newFirstCapturedUpdate = null;
  update = queue.firstCapturedUpdate;
  while (update !== null) {
    var _updateExpirationTime = update.expirationTime;
    if (_updateExpirationTime < renderExpirationTime) {
      // This update does not have sufficient priority. Skip it.
      if (newFirstCapturedUpdate === null) {
        // This is the first skipped captured update. It will be the first
        // update in the new list.
        newFirstCapturedUpdate = update;
        // 碰到第一个跳过的update时，newBaseState记录下此时的resultState
        if (newFirstUpdate === null) {
          newBaseState = resultState;
        }
      }
      // Since this update will remain in the list, update the remaining
      // expiration time.
      if (newExpirationTime < _updateExpirationTime) {
        newExpirationTime = _updateExpirationTime;
      }
    } else {
      // This update does have sufficient priority. Process it and compute
      // a new result.
      resultState = getStateFromUpdate(workInProgress, queue, update, resultState, props, instance);
      var _callback2 = update.callback;
      if (_callback2 !== null) {
        workInProgress.effectTag |= Callback;
        // Set this to null, in case it was mutated during an aborted render.
        update.nextEffect = null;
        if (queue.lastCapturedEffect === null) {
          queue.firstCapturedEffect = queue.lastCapturedEffect = update;
        } else {
          queue.lastCapturedEffect.nextEffect = update;
          queue.lastCapturedEffect = update;
        }
      }
    }
    update = update.next;
  }
  //没有跳过的update时，清空lastUpdate
  if (newFirstUpdate === null) {
    queue.lastUpdate = null;
  }
  if (newFirstCapturedUpdate === null) {
    queue.lastCapturedUpdate = null;
  } else {
    workInProgress.effectTag |= Callback;
  }
  //当newFirstUpdate不存在的时候，就是没有跳过的update时，最终状态赋值给newBaseState，如果有跳过的update，newBaseState是跳过之前的state.
  if (newFirstUpdate === null && newFirstCapturedUpdate === null) {
    // We processed every update, without skipping. That means the new base
    // state is the same as the result state.
    newBaseState = resultState;
  }
  //更新属性
  queue.baseState = newBaseState;
  queue.firstUpdate = newFirstUpdate;
  queue.firstCapturedUpdate = newFirstCapturedUpdate;

  
  workInProgress.expirationTime = newExpirationTime;
  workInProgress.memoizedState = resultState;

  {
    currentlyProcessingQueue = null;
  }
}

// getStateFromUpdate 中的 prevState 就是 processUpdateQueue resultState；

重点看UpdateState
function getStateFromUpdate(workInProgress, queue, update, prevState, nextProps, instance) {
  switch (update.tag) {
    case ReplaceState:
      {
        var _payload = update.payload;
        if (typeof _payload === 'function') {
          // Updater function
          {
            enterDisallowedContextReadInDEV();
            if (debugRenderPhaseSideEffects || debugRenderPhaseSideEffectsForStrictMode && workInProgress.mode & StrictMode) {
              _payload.call(instance, prevState, nextProps);
            }
          }
          var nextState = _payload.call(instance, prevState, nextProps);
          {
            exitDisallowedContextReadInDEV();
          }
          return nextState;
        }
        // State object
        return _payload;
      }
    case CaptureUpdate:
      {
        workInProgress.effectTag = workInProgress.effectTag & ~ShouldCapture | DidCapture;
      }
    // Intentional fallthrough
    case UpdateState:
      {
        var _payload2 = update.payload;
        var partialState = void 0;
        //是函数就调用改函数，将结果赋值partialState
        if (typeof _payload2 === 'function') {
          // Updater function
          {
            enterDisallowedContextReadInDEV();
            if (debugRenderPhaseSideEffects || debugRenderPhaseSideEffectsForStrictMode && workInProgress.mode & StrictMode) {
              _payload2.call(instance, prevState, nextProps);
            }
          }
          //prevState 是上一个update计算的结果：resultState；。
          partialState = _payload2.call(instance, prevState, nextProps);
          {
            exitDisallowedContextReadInDEV();
          }
        } else {
          // Partial state object
          partialState = _payload2;
        }
        if (partialState === null || partialState === undefined) {
          // Null and undefined are treated as no-ops.
          return prevState;
        }
        // 将当前结果和prevState进行浅合并

        return _assign({}, prevState, partialState);
      }
    case ForceUpdate:
      {
        hasForceUpdate = true;
        return prevState;
      }
  }
  return prevState;
}
```


经过processUpdateQueue 处理，updateQueue中的update最终会被合并。这就是批量更新处理state的奥秘所在。


### scheduleWork中的requestWork

由于scheduleWork会调用requestWork，主要逻辑在requestWork中

- isRendering 在performWorkOnRoot开始设置为true,结束设置为false
- isBatchingUpdates 进入react合成事件时设置为true

isRendering或者isBatchingUpdates为true的时候会进入批量渲染，直接返回继续处理下一个setState。

```
function requestWork(root, expirationTime) {
  addRootToSchedule(root, expirationTime);
  if (isRendering) {
    // Prevent reentrancy. Remaining work will be scheduled at the end of
    // the currently rendering batch.
    return;
  }

  if (isBatchingUpdates) {
    // Flush work at the end of the batch.
    if (isUnbatchingUpdates) {
      // ...unless we're inside unbatchedUpdates, in which case we should
      // flush it now.
      nextFlushedRoot = root;
      nextFlushedExpirationTime = Sync;
      performWorkOnRoot(root, Sync, false);
    }
    return;
  }

  // TODO: Get rid of Sync and use current time?
  if (expirationTime === Sync) {
    performSyncWork();
  } else {
    scheduleCallbackWithExpirationTime(root, expirationTime);
  }
}

```

### 关于updateQueue和其优先级

UpdateQueue是一个有优先级的单链表结构

UpdateQueue和fiber一样是成对出现的，一个current queue代表屏幕上可见的状态，一个work-in-progress queue在提交之前是可以更改和异步处理的，如果work-in-progress在完成之前无效了，会克隆current queue来创建一个新的work-in-progress queue


两个队列共享一个持久的，单链接的列表结构。调度一个新的update时，会追加到两个队列的末尾，每个队列维护一个firstUpdate指针，指向尚未处理的update,work-in-progress 指针的位置总是大于或者等于当前更新的位置，所有的工作会在work-in-progress上进行。current queue 的指针只会在commit阶段被更新，会和work-in-progress进行交换。


```
/ For example:
//
//   Current pointer:           A - B - C - D - E - F
//   Work-in-progress pointer:              D - E - F
//                                          ^
//                                          The work-in-progress queue has
//                                          processed more updates than current.
```



> 同时追加到两个队列的原因是不这样处理可能会丢失一些update,比如我们只给work-in-progress queue增加update,当work-in-progress 重启的时候从current queue克隆时会丢失updates，同样的，只给current queue增加update,当work-in-progress queue提交时，会和current queue交换，此时update也会丢失。


###### Prioritization

update并不是按照优先级排序的，而是按照插入的顺序，新的update总是被插入到队列的结尾。

当在render 阶段处理update queue时,只有有有效优先级的update会被处理，无有效优先级的udate会被跳过，它会保留在队列当中并在之后进行处理。最重要的是，跳过的update之后的所有update都会保留在队列当中，这意味着高优先级的update会被处理两次，我们跟踪了一个baseState，代表跳过的update之前的state。

```
参考processUpdateQueue 方法来看
// For example:
//
//   Given a base state of '', and the following queue of updates
//
//     A1 - B2 - C1 - D2
//
//   where the number indicates the priority, and the update is applied to the
//   previous state by appending a letter, React will process these updates as
//   two separate renders, one per distinct priority level:
//
//   First render, at priority 1:
//     Base state: ''
//     Updates: [A1, C1]
//     Result state: 'AC'
//
//   Second render, at priority 2:
//     Base state: 'A'            <-  The base state does not include C1,
//                                    because B2 was skipped.
//     Updates: [B2, C1, D2]      <-  C1 was rebased on top of B2
//     Result state: 'ABCD'
//
```


这种处理方法始终能保持最终的state是正确的。





