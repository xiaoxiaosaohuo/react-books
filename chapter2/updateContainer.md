## updateContainer

前面提到这个方法在ReactRoot的render方法中调用

```
updateContainer(children, root, null, work._onCommit);
```

```
export function updateContainer(
  element: ReactNodeList,
  container: OpaqueRoot,
  parentComponent: ?React$Component<any, any>,
  callback: ?Function,
): ExpirationTime {
  const current = container.current;
  const currentTime = requestCurrentTime();
  const expirationTime = computeExpirationForFiber(currentTime, current);
  return updateContainerAtExpirationTime(
    element,
    container,
    parentComponent,
    expirationTime,
    callback,
  );
}

```

在该方法中，React首先会调用requestCurrentTime获取当前expirationTime， 然后调用computeExpirationForFiber来确定优先级，computeExpirationForFiber会根据当前工作类型，比如isWorking、 isCommitting等等来返回对应的优先级，首次渲染默认是Sync.

> expirationTime前面有介绍

根据上面得到的expirationTime（优先级），经过updateContainerAtExpirationTime -> scheduleRootUpdate


scheduleRootUpdate接下来就是React新版本异步渲染的核心了