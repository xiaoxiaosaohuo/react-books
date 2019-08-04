## reconcilation

在使用react的时候，render()方法会创建一个React elements tree，当state或者props更新时，render方法会再次调用返回一个新的React elements tree,React需要弄清楚如何有效地更新UI以匹配最新的树。
React Reconciliation是React的diff算法，该算法基于两大假设

- Two elements of different types will produce different trees.
- The developer can hint at which child elements may be stable across different renders with a key prop.
  
diff 的过程其实就是 reconciliation 的过程，在生成 workInProgress tree 的时候，新 fiber 的生成就取决于老 fiber 与相关 props。


在  `beginWork`方法中有很多`update***`的方法，这些方法最后会调用一个方法叫---- `reconcileChildren`



reconcileChildren 只是一个入口函数，如果首次渲染，current 空 null，就通过 mountChildFibers 创建子节点的 Fiber 实例。如果不是首次渲染，就调用 reconcileChildFibers去做 diff，然后得出 effect list。

```
export function reconcileChildren(
  current: Fiber | null,
  workInProgress: Fiber,
  nextChildren: any,
  renderExpirationTime: ExpirationTime,
) {
  if (current === null) {
    // If this is a fresh new component that hasn't been rendered yet, we
    // won't update its child set by applying minimal side-effects. Instead,
    // we will add them all to the child before it gets rendered. That means
    // we can optimize this reconciliation pass by not tracking side-effects.
    workInProgress.child = mountChildFibers(
      workInProgress,
      null,
      nextChildren,
      renderExpirationTime,
    );
  } else {
    // If the current child is the same as the work in progress, it means that
    // we haven't yet started any work on these children. Therefore, we use
    // the clone algorithm to create a copy of all the current children.

    // If we had any progressed work already, that is invalid at this point so
    // let's throw it out.
    workInProgress.child = reconcileChildFibers(
      workInProgress,
      current.child,
      nextChildren,
      renderExpirationTime,
    );
  }
}

```

mountChildFibers和reconcileChildFibers方法是一样的，他们都是调用`ChildReconciler`方法生成的，只是参数不同

```
export const reconcileChildFibers = ChildReconciler(true);
export const mountChildFibers = ChildReconciler(false);

```

- shouldTrackSideEffects,判断是否要增加一些effectTag用来优化初次渲染，因为初次渲染没有更新操作。





ChildReconciler包含很多方法

- deleteChild
- deleteRemainingChildren
- mapRemainingChildren
- useFiber
- placeChild
- placeSingleChild
- updateTextNode
- updateElement
- reconcileChildFibers
等等


## 入口方法reconcileChildFibers

- ChildReconciler最终返回reconcileChildFibers方法，作为reconlication的入口函数


- reconcileChildFibers 会根据newChild的不同类型进行不同的处理
- reconcileChildFibers最终的返回是当前节点的第一个孩子节点，会在performUnitWork中 return 并赋值给nextUnitOfWork
  

```
function reconcileChildFibers(
    returnFiber: Fiber,
    currentFirstChild: Fiber | null,
    newChild: any,
    expirationTime: ExpirationTime,
  ): Fiber | null {
    // REACT_FRAGMENT_TYPE
    const isUnkeyedTopLevelFragment =
      typeof newChild === 'object' &&
      newChild !== null &&
      newChild.type === REACT_FRAGMENT_TYPE &&
      newChild.key === null;
    if (isUnkeyedTopLevelFragment) {
      newChild = newChild.props.children;
    }

    // Handle object types
    const isObject = typeof newChild === 'object' && newChild !== null;

    if (isObject) {
      switch (newChild.$$typeof) {
        case REACT_ELEMENT_TYPE:
          return placeSingleChild(
            reconcileSingleElement(
              returnFiber,
              currentFirstChild,
              newChild,
              expirationTime,
            ),
          );
        case REACT_PORTAL_TYPE:
          return placeSingleChild(
            reconcileSinglePortal(
              returnFiber,
              currentFirstChild,
              newChild,
              expirationTime,
            ),
          );
      }
    }

    if (typeof newChild === 'string' || typeof newChild === 'number') {
      return placeSingleChild(
        reconcileSingleTextNode(
          returnFiber,
          currentFirstChild,
          '' + newChild,
          expirationTime,
        ),
      );
    }

    if (isArray(newChild)) {
      return reconcileChildrenArray(
        returnFiber,
        currentFirstChild,
        newChild,
        expirationTime,
      );
    }

    if (getIteratorFn(newChild)) {
      return reconcileChildrenIterator(
        returnFiber,
        currentFirstChild,
        newChild,
        expirationTime,
      );
    }

    if (isObject) {
      throwOnInvalidObjectType(returnFiber, newChild);
    }

    if (typeof newChild === 'undefined' && !isUnkeyedTopLevelFragment) {
      // If the new child is undefined, and the return fiber is a composite
      // component, throw an error. If Fiber return types are disabled,
      // we already threw above.
      switch (returnFiber.tag) {
        case ClassComponent: {
          if (__DEV__) {
            const instance = returnFiber.stateNode;
            if (instance.render._isMockFunction) {
              // We allow auto-mocks to proceed as if they're returning null.
              break;
            }
          }
        }
        case FunctionComponent: {
          const Component = returnFiber.type;
        }
      }
    }

    // Remaining cases are all treated as empty.
    return deleteRemainingChildren(returnFiber, currentFirstChild);
  }
```