<!--
 * @Description: In User Settings Edit
 * @Author: your name
 * @Date: 2019-09-05 23:10:56
 * @LastEditTime: 2019-09-08 18:30:33
 * @LastEditors: Please set LastEditors
 -->
## render和commit阶段的异常处理

*** 这里的render和commit是值React Fiber架构的两个工作阶段。***


在`ReactFiberThrow.js`中有几个处理fiber异常的方法，下面将介绍。


<!-- throwException 方法主要是处理render阶段(第二阶段是commit)的错误和异常。 -->

我们知道在render阶段有个workLoop大循环，其简化的结构如下。
可以看到，当workLoop中的错误将在这里被catch到

```
do {
    try {
      workLoop(isYieldy);
    } catch (thrownValue) {
        //错误捕获逻辑
    }
}while(true)
```


在catch中有两点重要的逻辑


```
// 第一点
if (true && replayFailedUnitOfWorkWithInvokeGuardedCallback) {
    if (mayReplay) {
    var failedUnitOfWork = nextUnitOfWork;
    replayUnitOfWork(failedUnitOfWork, thrownValue, isYieldy);
    }
}

// 第二点
var sourceFiber = nextUnitOfWork;
var returnFiber = sourceFiber.return;
if (returnFiber === null) {
    didFatal = true;
    onUncaughtError$1(thrownValue);
} else {
    throwException(root, returnFiber, sourceFiber, thrownValue, nextRenderExpirationTime);
    nextUnitOfWork = completeUnitOfWork(sourceFiber);
    continue;
}
```
#### 第一点 
replayUnitOfWork 方法，从其名字可以看出是重新运行工作

参数：

-failedUnitOfWork 表示当前有异常的工作单元、
- thrownValue 异常信息
- isYieldy 

replayUnitOfWork 的核心作用是重新运行一遍刚才的workLoop，相当于调用 `performUnitOfWork(nextUnitOfWork)`,其主要目的是搜集详细的错误堆栈信息。这一步也是通过调用`invokeGuardedCallback` 实现的。

```
 invokeGuardedCallback(null, workLoop, null, isYieldy);
```

#### 第二点


这里有两种异常，一种是根节点的，一种是其他节点的

- RootError，最后是调用 onUncaughtError 函数处理
- ClassError，最后是调用 throwException函数处理错误和completeUnitOfWork完成剩余工作


#### throwException 


```
function throwException(root, returnFiber, sourceFiber, value, renderExpirationTime) {
  // 当前fiber 标记为 Incomplete

  sourceFiber.effectTag |= Incomplete;
 // 删除当前fiber的 effect list
  sourceFiber.firstEffect = sourceFiber.lastEffect = null;
    // throw value是一个promise ，这是suspend组件的错误
  if (value !== null && typeof value === 'object' && typeof value.then === 'function') {
    var thenable = value;

    // Find the earliest timeout threshold of all the placeholders in the
    // ancestor path. We could avoid this traversal by storing the thresholds on
    // the stack, but we choose not to because we only hit this path if we're
    // IO-bound (i.e. if something suspends). Whereas the stack is used even in
    // the non-IO- bound case.
    // 向上遍历，在祖先中找到最早过期的
    var _workInProgress = returnFiber;
    var earliestTimeoutMs = -1;
    var startTimeMs = -1;
    do {
      if (_workInProgress.tag === SuspenseComponent) {
        var current$$1 = _workInProgress.alternate;
        if (current$$1 !== null) {
          var currentState = current$$1.memoizedState;
          if (currentState !== null) {
            var timedOutAt = currentState.timedOutAt;
            startTimeMs = expirationTimeToMs(timedOutAt);
            break;
          }
        }
        var defaultSuspenseTimeout = 150;
        if (earliestTimeoutMs === -1 || defaultSuspenseTimeout < earliestTimeoutMs) {
          earliestTimeoutMs = defaultSuspenseTimeout;
        }
      }
      _workInProgress = _workInProgress.return;
    } while (_workInProgress !== null);

    // 调度最近的Suspense组件，重新渲染 timed out 视图
    _workInProgress = returnFiber;
    do {
      if (_workInProgress.tag === SuspenseComponent && shouldCaptureSuspense(_workInProgress)) {
        var thenables = _workInProgress.updateQueue;
        if (thenables === null) {
          var updateQueue = new Set();
          updateQueue.add(thenable);
          _workInProgress.updateQueue = updateQueue;
        } else {
          thenables.add(thenable);
        }

        if ((_workInProgress.mode & ConcurrentMode) === NoEffect) {
          _workInProgress.effectTag |= DidCapture;

          sourceFiber.effectTag &= ~(LifecycleEffectMask | Incomplete);

          if (sourceFiber.tag === ClassComponent) {
            var currentSourceFiber = sourceFiber.alternate;
            if (currentSourceFiber === null) {
              sourceFiber.tag = IncompleteClassComponent;
            } else {
              var update = createUpdate(Sync);
              update.tag = ForceUpdate;
              enqueueUpdate(sourceFiber, update);
            }
          }

          sourceFiber.expirationTime = Sync;

          return;
        }


        attachPingListener(root, renderExpirationTime, thenable);

        var absoluteTimeoutMs = void 0;
        if (earliestTimeoutMs === -1) {
          absoluteTimeoutMs = maxSigned31BitInt;
        } else {
          if (startTimeMs === -1) {
            startTimeMs = inferStartTimeFromExpirationTime$$1(root, renderExpirationTime);
          }
          absoluteTimeoutMs = startTimeMs + earliestTimeoutMs;
        }

        renderDidSuspend$$1(root, absoluteTimeoutMs, renderExpirationTime);

        _workInProgress.effectTag |= ShouldCapture;
        _workInProgress.expirationTime = renderExpirationTime;
        return;
      } else if (enableSuspenseServerRenderer && _workInProgress.tag === DehydratedSuspenseComponent) {
        attachPingListener(root, renderExpirationTime, thenable);

        var retryCache = _workInProgress.memoizedState;
        if (retryCache === null) {
          retryCache = _workInProgress.memoizedState = new PossiblyWeakSet$1();
          var _current = _workInProgress.alternate;
          (function () {
            if (!_current) {
              {
                throw ReactError('A dehydrated suspense boundary must commit before trying to render. This is probably a bug in React.');
              }
            }
          })();
          _current.memoizedState = retryCache;
        }
        if (!retryCache.has(thenable)) {
          retryCache.add(thenable);
          var retry = resolveRetryThenable$$1.bind(null, _workInProgress, thenable);
          if (enableSchedulerTracing) {
            retry = unstable_wrap(retry);
          }
          thenable.then(retry, retry);
        }
        _workInProgress.effectTag |= ShouldCapture;
        _workInProgress.expirationTime = renderExpirationTime;
        return;
      }
      _workInProgress = _workInProgress.return;
    } while (_workInProgress !== null);
    value = new Error((getComponentName(sourceFiber.type) || 'A React component') + ' suspended while rendering, but no fallback UI was specified.\n' + '\n' + 'Add a <Suspense fallback=...> component higher in the tree to ' + 'provide a loading indicator or placeholder to display.' + getStackByFiberInDevAndProd(sourceFiber));
  }

  renderDidError$$1();
  value = createCapturedValue(value, sourceFiber);
  var workInProgress = returnFiber;
  do {
    switch (workInProgress.tag) {
      case HostRoot:
        {
          var _errorInfo = value;
          workInProgress.effectTag |= ShouldCapture;
          workInProgress.expirationTime = renderExpirationTime;
          var _update = createRootErrorUpdate(workInProgress, _errorInfo, renderExpirationTime);
          enqueueCapturedUpdate(workInProgress, _update);
          return;
        }
      case ClassComponent:
        // Capture and retry
        var errorInfo = value;
        var ctor = workInProgress.type;
        var instance = workInProgress.stateNode;
        if ((workInProgress.effectTag & DidCapture) === NoEffect && (typeof ctor.getDerivedStateFromError === 'function' || instance !== null && typeof instance.componentDidCatch === 'function' && !isAlreadyFailedLegacyErrorBoundary$$1(instance))) {
          workInProgress.effectTag |= ShouldCapture;
          workInProgress.expirationTime = renderExpirationTime;
          // Schedule the error boundary to re-render using updated state
          var _update2 = createClassErrorUpdate(workInProgress, errorInfo, renderExpirationTime);
          enqueueCapturedUpdate(workInProgress, _update2);
          return;
        }
        break;
      default:
        break;
    }
    workInProgress = workInProgress.return;
  } while (workInProgress !== null);
}
```

这个函数的重点是后半部分，会遍历当前异常节点的所有父节点，判断各节点的类型，主要是HostRoot || ClassComponent 这两种类型

> 值得注意的是 异常边界组件自身的异常不会捕获并处理,所以要向上遍历


- HostRoot ,直接调用`createRootErrorUpdate`创建update，并将此update放入此节点的异常更新队列中
- ClassComponent,会判断是否是异常边界组件，即是否存在componentDidCatch，或者getDerivedStateFromError 方法。若当前classCompoennt是一个异常边界组件，那么就调用`createClassErrorUpdate ` 创建update,将update放入异常更新队列中

#### createRootErrorUpdate，createClassErrorUpdate
```
function createRootErrorUpdate(fiber, errorInfo, expirationTime) {
  var update = createUpdate(expirationTime);
  // 通过返回null卸载root
  update.tag = CaptureUpdate;
  update.payload = { element: null };
  var error = errorInfo.value;
  // callback 将在commitUpdateQueue方法中调用
  update.callback = function () {
    onUncaughtError$$1(error);
    logError(fiber, errorInfo);
  };
  return update;
}

function createClassErrorUpdate(fiber, errorInfo, expirationTime) {
  var update = createUpdate(expirationTime);
  // 标记update的tag
  update.tag = CaptureUpdate;
  // 调用getDerivedStateFromError
  var getDerivedStateFromError = fiber.type.getDerivedStateFromError;
  if (typeof getDerivedStateFromError === 'function') {
    var error = errorInfo.value;
    update.payload = function () {
        // getDerivedStateFromError 用于渲染错误的UI视图
      return getDerivedStateFromError(error);
    };
  }

  var inst = fiber.stateNode;
  if (inst !== null && typeof inst.componentDidCatch === 'function') {
    update.callback = function callback() {
      if (typeof getDerivedStateFromError !== 'function') {
        markLegacyErrorBoundaryAsFailed$$1(this);
      }
      var error = errorInfo.value;
      var stack = errorInfo.stack;
      logError(fiber, errorInfo);
      this.componentDidCatch(error, {
        componentStack: stack !== null ? stack : ''
      });
    };
  }
  return update;
}

//  设置hasUnhandledError 变量 和unhandledError变量，打印错误。
function onUncaughtError$1(error) {
  (function () {
    if (!(nextFlushedRoot !== null)) {
      {
        throw ReactError('Should be working on a root. This error is likely caused by a bug in React. Please file an issue.');
      }
    }
  })();
  nextFlushedRoot.expirationTime = NoWork;
  if (!hasUnhandledError) {
    hasUnhandledError = true;
    unhandledError = error;
  }
}
```


#### unwindWork

在completeUnitOfWork时，如果当前工作是`Incomplete` 就会调用 `unwindWork` 方法将当前fiber弹出fiber stack

其返回值都是null

```
function unwindWork(workInProgress, renderExpirationTime) {
  switch (workInProgress.tag) {
    case ClassComponent:
      {
        var Component = workInProgress.type;
        if (isContextProvider(Component)) {
          popContext(workInProgress);
        }
        var effectTag = workInProgress.effectTag;
        if (effectTag & ShouldCapture) {
          workInProgress.effectTag = effectTag & ~ShouldCapture | DidCapture;
          return workInProgress;
        }
        return null;
      }
    case HostRoot:
      {
        popHostContainer(workInProgress);
        popTopLevelContextObject(workInProgress);
        var _effectTag = workInProgress.effectTag;
        (function () {
          if (!((_effectTag & DidCapture) === NoEffect)) {
            {
              throw ReactError('The root failed to unmount after an error. This is likely a bug in React. Please file an issue.');
            }
          }
        })();
        workInProgress.effectTag = _effectTag & ~ShouldCapture | DidCapture;
        return workInProgress;
      }
    case HostComponent:
      {
        // TODO: popHydrationState
        popHostContext(workInProgress);
        return null;
      }
    case SuspenseComponent:
      {
        var _effectTag2 = workInProgress.effectTag;
        if (_effectTag2 & ShouldCapture) {
          workInProgress.effectTag = _effectTag2 & ~ShouldCapture | DidCapture;
          // Captured a suspense effect. Re-render the boundary.
          return workInProgress;
        }
        return null;
      }
    case DehydratedSuspenseComponent:
      {
        if (enableSuspenseServerRenderer) {
          // TODO: popHydrationState
          var _effectTag3 = workInProgress.effectTag;
          if (_effectTag3 & ShouldCapture) {
            workInProgress.effectTag = _effectTag3 & ~ShouldCapture | DidCapture;
            // Captured a suspense effect. Re-render the boundary.
            return workInProgress;
          }
        }
        return null;
      }
    case HostPortal:
      popHostContainer(workInProgress);
      return null;
    case ContextProvider:
      popProvider(workInProgress);
      return null;
    case EventComponent:
    case EventTarget:
      if (enableEventAPI) {
        popHostContext(workInProgress);
      }
      return null;
    default:
      return null;
  }
}
```


#### commit 阶段的错误

commit阶段的错误主要是集中在三个大循环当中，也是通过调用`invokeGuardedCallback` 来捕获的。


#### finishRendering

在performWork的最后阶段会调用 

```

function performWork(minExpirationTime) {
  findHighestPriorityRoot();

  while (nextFlushedRoot !== null && nextFlushedExpirationTime !== NoWork && minExpirationTime <= nextFlushedExpirationTime) {
    performWorkOnRoot(nextFlushedRoot, nextFlushedExpirationTime, false);
    findHighestPriorityRoot();
  }

  if (nextFlushedExpirationTime !== NoWork) {
    scheduleCallbackWithExpirationTime(nextFlushedRoot, nextFlushedExpirationTime);
  }

  // 清理工作
  finishRendering();
}


function finishRendering() {
  nestedUpdateCount = 0;
  lastCommittedRootDuringThisBatch = null;

  {
    if (rootWithPendingPassiveEffects === null) {
      nestedPassiveEffectCountDEV = 0;
    }
  }

  if (completedBatches !== null) {
    var batches = completedBatches;
    completedBatches = null;
    for (var i = 0; i < batches.length; i++) {
      var batch = batches[i];
      try {
        batch._onComplete();
      } catch (error) {
        if (!hasUnhandledError) {
          hasUnhandledError = true;
          unhandledError = error;
        }
      }
    }
  }
    // 有错误就throw,重置错误变量
  if (hasUnhandledError) {
    var error = unhandledError;
    unhandledError = null;
    hasUnhandledError = false;
    throw error;
  }
}
```