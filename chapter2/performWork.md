## performWork

performWork分为同步和异步两种。同步时``` performWork(Sync, false)``` 异步时 ``` performWork(NoWork, true)```

```
function performWork(minExpirationTime, isYieldy) {
  // Keep working on roots until there's no more work, or until there's a higher
  // priority event.
  findHighestPriorityRoot();

  if (isYieldy) {
    recomputeCurrentRendererTime();
    currentSchedulerTime = currentRendererTime;

    if (enableUserTimingAPI) {
      var didExpire = nextFlushedExpirationTime > currentRendererTime;
      var timeout = expirationTimeToMs(nextFlushedExpirationTime);
      stopRequestCallbackTimer(didExpire, timeout);
    }
    // nextFlushedRoot !== null  nextFlushedRoot 下一个将要执行的 FiberRoot
    // nextFlushedExpirationTime !== NoWork 下一个FiberRoot中还有未执行的任务
    // minExpirationTime <= nextFlushedExpirationTime 表示当前任务队列中还有同步任务需要执行
    // currentRendererTime >= nextFlushedExpirationTime 下一个任务已经超时未更新了
    while (
        nextFlushedRoot !== null &&
        nextFlushedExpirationTime !== NoWork && 
        minExpirationTime <= nextFlushedExpirationTime &&
        !(didYield && currentRendererTime > nextFlushedExpirationTime
         )) {
      performWorkOnRoot(nextFlushedRoot, nextFlushedExpirationTime, currentRendererTime > nextFlushedExpirationTime);
      findHighestPriorityRoot();
      recomputeCurrentRendererTime();
      currentSchedulerTime = currentRendererTime;
    }
  } else {
    while (nextFlushedRoot !== null && nextFlushedExpirationTime !== NoWork && minExpirationTime <= nextFlushedExpirationTime) {
      performWorkOnRoot(nextFlushedRoot, nextFlushedExpirationTime, false);
      findHighestPriorityRoot();
    }
  }

  // We're done flushing work. Either we ran out of time in this callback,
  // or there's no more work left with sufficient priority.

  // If we're inside a callback, set this to false since we just completed it.
  if (isYieldy) {
    callbackExpirationTime = NoWork;
    callbackID = null;
  }
  // If there's work left over, schedule a new callback.
  if (nextFlushedExpirationTime !== NoWork) {
    scheduleCallbackWithExpirationTime(nextFlushedRoot, nextFlushedExpirationTime);
  }

  // Clean-up.
  finishRendering();
}
```

*** findHighestPriorityRoot *** 函数主要执行两个操作， 一个是判断当前root是否还有任务，如果没有， 则从firstScheuleRoot链中移除。 一个是找出优先级最高的root和其对应的优先级并赋值给
nextFlushedRoot和nextFlushedExpirationTime

```
function findHighestPriorityRoot() {
  var highestPriorityWork = NoWork;
  var highestPriorityRoot = null;
  if (lastScheduledRoot !== null) {
    var previousScheduledRoot = lastScheduledRoot;
    var root = firstScheduledRoot;
    while (root !== null) {
      var remainingExpirationTime = root.expirationTime;
      if (remainingExpirationTime === NoWork) {
         // 判断是否还有任务并移除
      } else {
         // 找出最高的优先级root和其对应的优先级
      }
    }
  }
  // 赋值
  nextFlushedRoot = highestPriorityRoot;
  nextFlushedExpirationTime = highestPriorityWork;
}
```


## performWorkOnRoot


performWorkOnRoot传入优先级最高的root和其对应的expirationTime以及一个isYieldy作为参数

isYieldy用判断是否可以中断


```
function performWorkOnRoot(root, expirationTime, isYieldy) {
  
  isRendering = true;

  //检查是否是异步任务或者是同步/过期任务
  if (!isYieldy) {
    var finishedWork = root.finishedWork;
    if (finishedWork !== null) {
      // root已经完成了，可以进行commit.
      completeRoot(root, finishedWork, expirationTime);
    } else {
      root.finishedWork = null;
      //如果这个root之前已经suspended，就清空其timeoutHandle，准备重新渲染
      var timeoutHandle = root.timeoutHandle;
      if (timeoutHandle !== noTimeout) {
        root.timeoutHandle = noTimeout;
        cancelTimeout(timeoutHandle);
      }
      renderRoot(root, isYieldy);
      finishedWork = root.finishedWork;
      if (finishedWork !== null) {
        // root已经完成了，可以进行commit.
        completeRoot(root, finishedWork, expirationTime);
      }
    }
  } else {
    // 异步工作
    var _finishedWork = root.finishedWork;
    if (_finishedWork !== null) {
      // root已经完成了，可以进行commit.
      completeRoot(root, _finishedWork, expirationTime);
    } else {
      root.finishedWork = null;
      //如果这个root之前已经suspended，就清空其timeoutHandle，准备重新渲染
      var _timeoutHandle = root.timeoutHandle;
      if (_timeoutHandle !== noTimeout) {
        root.timeoutHandle = noTimeout;
        cancelTimeout(_timeoutHandle);
      }
      renderRoot(root, isYieldy);
      _finishedWork = root.finishedWork;
      if (_finishedWork !== null) {
        //root已经完成了，在commit前再次检查是否需要中断
        if (!shouldYieldToRenderer()) {
          // 还有剩余时间，commit root.
          completeRoot(root, _finishedWork, expirationTime);
        } else {
          // 没有剩余时间 ，标记root为已完成.之后再commit
          root.finishedWork = _finishedWork;
        }
      }
    }
  }

  isRendering = false;
}
```

renderRoot的产物会挂载到root的finishWork属性上， 首先performWorkOnRoot会先判断root的finishWork是否不为空， 如果存在的话则直接进入commit的阶段， 否则进入到renderRoot函数， 设置finishWork属性
renderRoot有2个参数，  ``` renderRoot(root, isYieldy) ```， 同步状态下isYield的值是false.


