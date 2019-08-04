## updateFunctionalComponent

- FunctionalComponent 就是一个函数，会通过调用`renderWithHooks`方法返回nextChildren
- 调用reconcileChildren

更多内容可以看 `Hooks 和 State Hook`章节

```
function updateFunctionComponent(
  current,
  workInProgress,
  Component,
  nextProps: any,
  renderExpirationTime,
) {
  

  const unmaskedContext = getUnmaskedContext(workInProgress, Component, true);
  const context = getMaskedContext(workInProgress, unmaskedContext);

  let nextChildren;
  prepareToReadContext(workInProgress, renderExpirationTime);
  if (enableFlareAPI) {
    prepareToReadListenerHooks(workInProgress);
  }
  nextChildren = renderWithHooks(
    current,
    workInProgress,
    Component,
    nextProps,
    context,
    renderExpirationTime,
  );

  if (current !== null && !didReceiveUpdate) {
    bailoutHooks(current, workInProgress, renderExpirationTime);
    return bailoutOnAlreadyFinishedWork(
      current,
      workInProgress,
      renderExpirationTime,
    );
  }

  // React DevTools reads this flag.
  workInProgress.effectTag |= PerformedWork;
  reconcileChildren(
    current,
    workInProgress,
    nextChildren,
    renderExpirationTime,
  );
  return workInProgress.child;
}
```

## renderWithHooks

- 给变量赋值 如`currentlyRenderingFiber`,`nextCurrentHook`等，主要是处理hooks
- 调用函数返回children
- 将之前的变量reset

```
export function renderWithHooks(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: any,
  props: any,
  refOrContext: any,
  nextRenderExpirationTime: ExpirationTime,
): any {
  renderExpirationTime = nextRenderExpirationTime;
  currentlyRenderingFiber = workInProgress;
  nextCurrentHook = current !== null ? current.memoizedState : null;


  ReactCurrentDispatcher.current =
  nextCurrentHook === null
    ? HooksDispatcherOnMount
    : HooksDispatcherOnUpdate;

  let children = Component(props, refOrContext);

  if (didScheduleRenderPhaseUpdate) {
    do {
      didScheduleRenderPhaseUpdate = false;
      numberOfReRenders += 1;

      // Start over from the beginning of the list
      nextCurrentHook = current !== null ? current.memoizedState : null;
      nextWorkInProgressHook = firstWorkInProgressHook;

      currentHook = null;
      workInProgressHook = null;
      componentUpdateQueue = null;

      ReactCurrentDispatcher.current = __DEV__
        ? HooksDispatcherOnUpdateInDEV
        : HooksDispatcherOnUpdate;

      children = Component(props, refOrContext);
    } while (didScheduleRenderPhaseUpdate);

    renderPhaseUpdates = null;
    numberOfReRenders = 0;
  }

  // We can assume the previous dispatcher is always this one, since we set it
  // at the beginning of the render phase and there's no re-entrancy.
  ReactCurrentDispatcher.current = ContextOnlyDispatcher;

  const renderedWork: Fiber = (currentlyRenderingFiber: any);

  renderedWork.memoizedState = firstWorkInProgressHook;
  renderedWork.expirationTime = remainingExpirationTime;
  renderedWork.updateQueue = (componentUpdateQueue: any);
  renderedWork.effectTag |= sideEffectTag;

  const didRenderTooFewHooks =
    currentHook !== null && currentHook.next !== null;

  renderExpirationTime = NoWork;
  currentlyRenderingFiber = null;

  currentHook = null;
  nextCurrentHook = null;
  firstWorkInProgressHook = null;
  workInProgressHook = null;
  nextWorkInProgressHook = null;

  remainingExpirationTime = NoWork;
  componentUpdateQueue = null;
  sideEffectTag = 0;

  // These were reset above
  // didScheduleRenderPhaseUpdate = false;
  // renderPhaseUpdates = null;
  // numberOfReRenders = 0;

  return children;
}

export function bailoutHooks(
  current: Fiber,
  workInProgress: Fiber,
  expirationTime: ExpirationTime,
) {
  workInProgress.updateQueue = current.updateQueue;
  workInProgress.effectTag &= ~(PassiveEffect | UpdateEffect);
  if (current.expirationTime <= expirationTime) {
    current.expirationTime = NoWork;
  }
}
```