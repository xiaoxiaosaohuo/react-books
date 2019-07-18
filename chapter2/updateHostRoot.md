## updateHostRoot

在scheduleRootUpdate的时候会为root创建update，包括的最主要信息是payload中的element,在updateHostRoot方法中会对update进行处理

```
function updateHostRoot(current$$1, workInProgress, renderExpirationTime) {

  pushHostRootContext(workInProgress);
  var updateQueue = workInProgress.updateQueue;
  var nextProps = workInProgress.pendingProps;
  var prevState = workInProgress.memoizedState;
  var prevChildren = prevState !== null ? prevState.element : null;

  processUpdateQueue(workInProgress, updateQueue, nextProps, null, renderExpirationTime);
  var nextState = workInProgress.memoizedState;
 
  var nextChildren = nextState.element;
  if (nextChildren === prevChildren) {
    // 前后无变化直接调用bailoutOnAlreadyFinishedWork
    resetHydrationState();
    return bailoutOnAlreadyFinishedWork(current$$1, workInProgress, renderExpirationTime);
  }
  var root = workInProgress.stateNode;
  //这个是hydrate相关的
  if ((current$$1 === null || current$$1.child === null) && root.hydrate && enterHydrationState(workInProgress)) {
    
    workInProgress.child = mountChildFibers(workInProgress, null, nextChildren, renderExpirationTime);
  } else {
    // Otherwise reset hydration state in case we aborted and resumed another
    // root.
    reconcileChildren(current$$1, workInProgress, nextChildren, renderExpirationTime);
    resetHydrationState();
  }
  return workInProgress.child;
}
```




``` ensureWorkInProgressQueueIsAClone ``` 确保workInProgress的UpdateQueue是current的一个拷贝，不在current上直接操作

``` processUpdateQueue ``` 处理UpdateQueue，计算出新的state，更新UpdateQueue的baseState，memoizedState。

``` reconcileChildren ``` 调和子节点，返回 workInProgress.child;