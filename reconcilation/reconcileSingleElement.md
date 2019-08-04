## 单个子节点

#### reconcileSingleElement

参数介绍
- returnFiber 父级fiber
- currentFirstChild  currentFiber第一个子节点
- element  reactElement,是调用render方法返回的nextChildren

主要流程

1. child 为null,表明是第一次挂载，直接创建fiber节点
2. child不为null时，遍历child的链表，找到第一个key和节点类型相同的节点，直接复用这个节点，然后删掉该节点以后的剩余child
3. 遍历的时候，如果key不同，就直接删除
```
function reconcileSingleElement(
    returnFiber: Fiber,
    currentFirstChild: Fiber | null,
    element: ReactElement,
    expirationTime: ExpirationTime,
  ): Fiber {
    const key = element.key;
    let child = currentFirstChild;
    while (child !== null) {
      // TODO: If key === null and child.key === null, then this only applies to
      // the first item in the list.
      if (child.key === key) {
          // key 相同，还要比较type
        if (
          child.tag === Fragment
            ? element.type === REACT_FRAGMENT_TYPE
            : child.elementType === element.type ||
              // Keep this check inline so it only runs on the false path:
              (__DEV__
                ? isCompatibleFamilyForHotReloading(child, element)
                : false)
        ) {
            // 因为diff的是单个的Element,所以要删除兄弟节点
          deleteRemainingChildren(returnFiber, child.sibling);
           // 基于child复制一个fiber，用传入的props,修改其pendingProps
          const existing = useFiber(
            child,
            element.type === REACT_FRAGMENT_TYPE
              ? element.props.children
              : element.props,
            expirationTime,
          );
          existing.ref = coerceRef(returnFiber, child, element);
          existing.return = returnFiber;
          if (__DEV__) {
            existing._debugSource = element._source;
            existing._debugOwner = element._owner;
          }
          return existing;
        } else {
               // type不同，删除该child及其兄弟节点
          deleteRemainingChildren(returnFiber, child);
          break;
        }
      } else {
        deleteChild(returnFiber, child);
      }
      child = child.sibling;
    }

    if (element.type === REACT_FRAGMENT_TYPE) {
      const created = createFiberFromFragment(
        element.props.children,
        returnFiber.mode,
        expirationTime,
        element.key,
      );
      created.return = returnFiber;
      return created;
    } else {
      const created = createFiberFromElement(
        element,
        returnFiber.mode,
        expirationTime,
      );
      created.ref = coerceRef(returnFiber, currentFirstChild, element);
      created.return = returnFiber;
      return created;
    }
  }
```



#### reconcileSingleTextNode

- 如果老节点是文字节点，就复用
- 如果老节点不是蚊子节点，删掉全部的老节点，创建一个新的文字节点
```
function reconcileSingleTextNode(
    returnFiber: Fiber,
    currentFirstChild: Fiber | null,
    textContent: string,
    expirationTime: ExpirationTime,
  ): Fiber {
    // There's no need to check for keys on text nodes since we don't have a
    // way to define them.
    if (currentFirstChild !== null && currentFirstChild.tag === HostText) {
      // We already have an existing node so let's just update it and delete
      // the rest.
      deleteRemainingChildren(returnFiber, currentFirstChild.sibling);
      const existing = useFiber(currentFirstChild, textContent, expirationTime);
      existing.return = returnFiber;
      return existing;
    }
    // The existing first child is not a text node so we need to create one
    // and delete the existing ones.
    deleteRemainingChildren(returnFiber, currentFirstChild);
    const created = createFiberFromText(
      textContent,
      returnFiber.mode,
      expirationTime,
    );
    created.return = returnFiber;
    return created;
  }
```
#### reconcileSinglePortal


```
function reconcileSinglePortal(
    returnFiber: Fiber,
    currentFirstChild: Fiber | null,
    portal: ReactPortal,
    expirationTime: ExpirationTime,
  ): Fiber {
    const key = portal.key;
    let child = currentFirstChild;
    while (child !== null) {
      // TODO: If key === null and child.key === null, then this only applies to
      // the first item in the list.
      if (child.key === key) {
        if (
          child.tag === HostPortal &&
          child.stateNode.containerInfo === portal.containerInfo &&
          child.stateNode.implementation === portal.implementation
        ) {
          deleteRemainingChildren(returnFiber, child.sibling);
          const existing = useFiber(
            child,
            portal.children || [],
            expirationTime,
          );
          existing.return = returnFiber;
          return existing;
        } else {
          deleteRemainingChildren(returnFiber, child);
          break;
        }
      } else {
        deleteChild(returnFiber, child);
      }
      child = child.sibling;
    }

    const created = createFiberFromPortal(
      portal,
      returnFiber.mode,
      expirationTime,
    );
    created.return = returnFiber;
    return created;
  }
```


#### useFiber 复用fiber的方法

```
function useFiber(
    fiber: Fiber,
    pendingProps: mixed,
    expirationTime: ExpirationTime,
  ): Fiber {
    // We currently set sibling to null and index to 0 here because it is easy
    // to forget to do before returning it. E.g. for the single child case.
    const clone = createWorkInProgress(fiber, pendingProps, expirationTime);
    clone.index = 0;
    clone.sibling = null;
    return clone;
  }
```

####  deleteRemainingChildren
删除 children 中剩余的节点，不是真的删除，只是打标记
```

  function deleteRemainingChildren(
    returnFiber: Fiber,
    currentFirstChild: Fiber | null,
  ): null {
    if (!shouldTrackSideEffects) {
      // Noop.
      return null;
    }

    // TODO: For the shouldClone case, this could be micro-optimized a bit by
    // assuming that after the first child we've already added everything.
    let childToDelete = currentFirstChild;
    while (childToDelete !== null) {
      deleteChild(returnFiber, childToDelete);
      childToDelete = childToDelete.sibling;
    }
    return null;
  }
```