## updateHostComponent

hostComponent指的是原生的HTML标签。
```
function updateHostComponent(current, workInProgress, renderExpirationTime) {
  pushHostContext(workInProgress);

  if (current === null) {
    tryToClaimNextHydratableInstance(workInProgress);
  }

  const type = workInProgress.type;
  const nextProps = workInProgress.pendingProps;
  const prevProps = current !== null ? current.memoizedProps : null;

  let nextChildren = nextProps.children;
  //判断是否是文字节点
  const isDirectTextChild = shouldSetTextContent(type, nextProps);

  if (isDirectTextChild) {
    
    nextChildren = null;
  } else if (prevProps !== null && shouldSetTextContent(type, prevProps)) {
      //从文字节点切换到正常节点后或者是空节点，加上ContentReset标记
    workInProgress.effectTag |= ContentReset;
  }

  markRef(current, workInProgress);

  // 检查属性中是否有offscreen/hidden.
  if (
    workInProgress.mode & ConcurrentMode &&
    renderExpirationTime !== Never &&
    shouldDeprioritizeSubtree(type, nextProps)
  ) {
    if (enableSchedulerTracing) {
      markSpawnedWork(Never);
    }
    // 如果是offscreen/hidden，将其expirationTime设置Never,然后直接bailout
    workInProgress.expirationTime = workInProgress.childExpirationTime = Never;
    return null;
  }

  reconcileChildren(
    current,
    workInProgress,
    nextChildren,
    renderExpirationTime,
  );
  return workInProgress.child;
}
```