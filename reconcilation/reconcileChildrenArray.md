## 多个子节点

#### 什么情况下newChild会是Array

1. children是一个数组

```
function Com1() {
  return array.map(i => <span key={i}>{i}</span>);
}
```

2. child的父级为Fragment

```
function Com() {
  return (
    <>
      <span>1</span>
      <span>2</span>
      <span>3</span>
    </>
  );
}
```

#### React是如何处理多个节点的？

1. 逐个对比两个数组,尝试通过index的顺序以及对比key来复用fiber节点
2. 调用updateSlot对比index相同的child的key是否相同，如果是，返回该对象，如果不是，返回null
3. 找道第一个不相等的位子，就跳出循环
4. 如果newChildren遍历完毕，就删除旧链表的剩余节点
5. 如果旧的遍历完毕，就插入newChildren剩余节点
6. 如果以上都不满足，就创建一个map,保存所有没有匹配到的节点，然后新的数组根据key从这个 map 里面查找，如果有则复用，没有则新建。
7. 在遍历过程中会调用placeChild方法，用于判断节点的 effects 是否该设置为 Placement

```
function reconcileChildrenArray(
    returnFiber: Fiber,
    currentFirstChild: Fiber | null,
    newChildren: Array<*>,
    expirationTime: ExpirationTime,
  ): Fiber | null {
    
      //存储链表的第一个指针
    let resultingFirstChild: Fiber | null = null;
    //链表的当前指针
    let previousNewFiber: Fiber | null = null;
    //
    let oldFiber = currentFirstChild;
    let lastPlacedIndex = 0;
    let newIdx = 0;
    let nextOldFiber = null;
    for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
      if (oldFiber.index > newIdx) {
        nextOldFiber = oldFiber;
        oldFiber = null;
      } else {
        nextOldFiber = oldFiber.sibling;
      }
      //// 如果可以复用就返回复用的fiber节点 否则返回null
    // 根据key来判断是否能够复用
      const newFiber = updateSlot(
        returnFiber,
        oldFiber,
        newChildren[newIdx],
        expirationTime,
      );
      //newFiber为null,跳出循环
      if (newFiber === null) {
      
        if (oldFiber === null) {
          oldFiber = nextOldFiber;
        }
        break;
      }
      if (shouldTrackSideEffects) {
          //返回的newFiber是新创建的，没有复用
        if (oldFiber && newFiber.alternate === null) {
            //删除oldFiber
          deleteChild(returnFiber, oldFiber);
        }
      }
      // 判断节点的 effects 是否为 Placement
      lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);

      // 将复用的fiber节点链接成一个链表
      if (previousNewFiber === null) {
        // TODO: Move out of the loop. This only happens for the first run.
        resultingFirstChild = newFiber;
      } else {
        
        previousNewFiber.sibling = newFiber;
      }
      previousNewFiber = newFiber;
      oldFiber = nextOldFiber;
    }
    //遍历newChildren完毕，删除旧链表剩余节点
    if (newIdx === newChildren.length) {
      // We've reached the end of the new children. We can delete the rest.
      deleteRemainingChildren(returnFiber, oldFiber);
      return resultingFirstChild;
    }
    // oldFiber不存在，表明旧的children已经遍历完毕，但是newChildren存在，说明是新增，剩余的新数组的项就可以作为新的项直接插入进去了。
    if (oldFiber === null) {
      // 遍历newChildren 创建新的fiber，通过sibling关联
      for (; newIdx < newChildren.length; newIdx++) {
        const newFiber = createChild(
          returnFiber,
          newChildren[newIdx],
          expirationTime,
        );
        if (newFiber === null) {
          continue;
        }
        lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
        if (previousNewFiber === null) {
          // TODO: Move out of the loop. This only happens for the first run.
          resultingFirstChild = newFiber;
        } else {
          previousNewFiber.sibling = newFiber;
        }
        previousNewFiber = newFiber;
      }
      return resultingFirstChild;
    }


    // 创建一个map,保存所有没有匹配到的节点，然后新的数组根据key从这个 map 里面查找，如果有则复用，没有则新建。
    const existingChildren = mapRemainingChildren(returnFiber, oldFiber);

    // Keep scanning and use the map to restore deleted items as moves.
    //从上面挑出for循环的索引开始继续向后遍历
    for (; newIdx < newChildren.length; newIdx++) {
        // 从map中取出和newChild的key值相同的fiber，然后创建fiber，更新属性，
      const newFiber = updateFromMap(
        existingChildren,
        returnFiber,
        newIdx,
        newChildren[newIdx],
        expirationTime,
      );
      if (newFiber !== null) {
        if (shouldTrackSideEffects) {
          if (newFiber.alternate !== null) {
            // The new fiber is a work in progress, but if there exists a
            // current, that means that we reused the fiber. We need to delete
            // it from the child list so that we don't add it to the deletion
            // list.
            // 存在则删除map中的fiber
            existingChildren.delete(
              newFiber.key === null ? newIdx : newFiber.key,
            );
          }
        }
        // 判断是否移动
        lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
        if (previousNewFiber === null) {
          resultingFirstChild = newFiber;
        } else {
          previousNewFiber.sibling = newFiber;
        }
        previousNewFiber = newFiber;
      }
    }

    if (shouldTrackSideEffects) {
      // Any existing children that weren't consumed above were deleted. We need
      // to add them to the deletion list.
      existingChildren.forEach(child => deleteChild(returnFiber, child));
    }

    return resultingFirstChild;
  }
```

#### updateSlot

对比新老的 key 是否相同，来查看是否可以复用老的节点。

判定规则：
1. Text节点：复用（key都为undefined）
2. Element节点：判断key是否相同以及type是否一致
3. 数组节点：当初顶层Fragment来处理（key都为undefined）

判定结果：
- 如果符合规则则返回fiber节点，不符合返回null
- elementType不匹配的话返回的fiber是新的而不是复用，也没有alternate绑定的关系
```
function updateSlot(
    returnFiber: Fiber,
    oldFiber: Fiber | null,
    newChild: any,
    expirationTime: ExpirationTime,
  ): Fiber | null {
    // Update the fiber if the keys match, otherwise return null.

    const key = oldFiber !== null ? oldFiber.key : null;

    if (typeof newChild === 'string' || typeof newChild === 'number') {
      // 对于新的节点如果是 string 或者 number，那么都是没有 key 的，
      // 所有如果老的节点有 key 的话，就不能复用，直接返回 null。
     
      if (key !== null) {
        return null;
      }
       // 老的节点 key 为 null 的话，代表老的节点是文本节点，就可以复用
      return updateTextNode(
        returnFiber,
        oldFiber,
        '' + newChild,
        expirationTime,
      );
    }

    if (typeof newChild === 'object' && newChild !== null) {
      switch (newChild.$$typeof) {
        case REACT_ELEMENT_TYPE: {
          if (newChild.key === key) {
            if (newChild.type === REACT_FRAGMENT_TYPE) {
              return updateFragment(
                returnFiber,
                oldFiber,
                newChild.props.children,
                expirationTime,
                key,
              );
            }
            return updateElement(
              returnFiber,
              oldFiber,
              newChild,
              expirationTime,
            );
          } else {
            return null;
          }
        }
        case REACT_PORTAL_TYPE: {
          if (newChild.key === key) {
            return updatePortal(
              returnFiber,
              oldFiber,
              newChild,
              expirationTime,
            );
          } else {
            return null;
          }
        }
      }

      if (isArray(newChild) || getIteratorFn(newChild)) {
        if (key !== null) {
          return null;
        }

        return updateFragment(
          returnFiber,
          oldFiber,
          newChild,
          expirationTime,
          null,
        );
      }

      throwOnInvalidObjectType(returnFiber, newChild);
    }

    if (__DEV__) {
      if (typeof newChild === 'function') {
        warnOnFunctionType();
      }
    }

    return null;
  }
```

#### mapRemainingChildren

```
function mapRemainingChildren(returnFiber, currentFirstChild) {
    // Add the remaining children to a temporary map so that we can find them by
    // keys quickly. Implicit (null) keys get added to this set with their index
    var existingChildren = new Map();

    var existingChild = currentFirstChild;
    while (existingChild !== null) {
      if (existingChild.key !== null) {
        existingChildren.set(existingChild.key, existingChild);
      } else {
        existingChildren.set(existingChild.index, existingChild);
      }
      existingChild = existingChild.sibling;
    }
    return existingChildren;
  }
```
#### 判断数组元素是否应该被移除的逻辑

这里尽量采用向后插入节点的方式，而不是向前插入

举个例子：
```
prev：[0, 1, 2, 3, 4] 
next: [0, 4, 3, 1, 2]
```

1. lastPlacedIndex = 0; oldIndex = 0; return 0;
2. lastPlacedIndex = 0; oldIndex = 4; oldIndex>=lastPlacedIndex; return 4;
3. lastPlacedIndex = 4; oldIndex = 3; <; return lastPlacedIndex(4);
最终得出effect列表：0 4 不需要移动，其他的都需要移动

```
function placeChild(
  newFiber: Fiber,
  lastPlacedIndex: number,
  newIndex: number,
): number {
  newFiber.index = newIndex; // index按照element的顺序
  if (!shouldTrackSideEffects) {
    // Noop.
    return lastPlacedIndex; // 这样不是永远都是0吗
  }
  const current = newFiber.alternate;
  if (current !== null) {
    const oldIndex = current.index;
    if (oldIndex < lastPlacedIndex) {
      // This is a move.
      newFiber.effectTag = Placement;
      return lastPlacedIndex;
    } else {
      // >=
      // This item can stay in place.
      return oldIndex;
    }
  } else {
    // This is an insertion.
    newFiber.effectTag = Placement;
    return lastPlacedIndex;
  }
}
```

