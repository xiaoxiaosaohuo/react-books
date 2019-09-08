<!--
 * @Description: In User Settings Edit
 * @Author: your name
 * @Date: 2019-09-05 15:42:12
 * @LastEditTime: 2019-09-08 18:09:53
 * @LastEditors: Please set LastEditors
 -->
## commitRoot

commitRoot 是进入commit阶段的核心方法

### 准备阶段

##### 由于进入commit阶段不能被打断，所以会设置一些标志位

```
isWorking = true;
isCommitting$1 = true;
```

##### effect list 只包括他的children,不包括self,所以如果root fiber 有effect,需要将其加入到effect list的末尾，


```
var firstEffect = void 0;
  if (finishedWork.effectTag > PerformedWork) {
    if (finishedWork.lastEffect !== null) {
      finishedWork.lastEffect.nextEffect = finishedWork;
      firstEffect = finishedWork.firstEffect;
    } else {
      firstEffect = finishedWork;
    }
  } else {
    // There is no effect on the root.
    firstEffect = finishedWork.firstEffect;
  }

   nextEffect = firstEffect;
```

### 三个大循环


接下来就是effect的提交工作。主要是三个大循环
每次迭代都会在`invokeGuardedCallback`中运行目标函数，目的是为了捕获错误


#### 第一次循环

主要是调用`getSnapshotBeforeUpdate` 生命周期，该方法的返回值将作为 `componentDidUpdate`的第三个参数 `snapshot`

```
while (nextEffect !== null) {
    var didError = false;
    var error = void 0;
    {
      invokeGuardedCallback(null, commitBeforeMutationLifecycles, null);
      if (hasCaughtError()) {
        didError = true;
        error = clearCaughtError();
      }
    }
    if (didError) {
      (function () {
        if (!(nextEffect !== null)) {
          {
            throw ReactError('Should have next effect. This error is likely caused by a bug in React. Please file an issue.');
          }
        }
      })();
      captureCommitPhaseError$1(nextEffect, error);
      // Clean-up
      if (nextEffect !== null) {
        nextEffect = nextEffect.nextEffect;
      }
    }
  }

  function commitBeforeMutationLifecycles() {
    while (nextEffect !== null) {
        {
        setCurrentFiber(nextEffect);
        }

        var effectTag = nextEffect.effectTag;
        if (effectTag & Snapshot) {
        recordEffect();
        var current$$1 = nextEffect.alternate;
        commitBeforeMutationLifeCycles(current$$1, nextEffect);
        }

        nextEffect = nextEffect.nextEffect;
    }

    {
        resetCurrentFiber();
    }
}
```

commitBeforeMutationLifecycles 根据 nextEffect.effectTag 是否有 Snapshot 把 `nextEffect.alternate ` fiber 对象和 `nextEffect ` 传入commitBeforeMutationLifeCycles 执行。

commitBeforeMutationLifeCycles 只对ClassComponent进行处理，主要是调用getSnapshotBeforeUpdate方法获取snapshot

```
function commitBeforeMutationLifeCycles(current$$1, finishedWork) {
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case SimpleMemoComponent:
      {
        commitHookEffectList(UnmountSnapshot, NoEffect$1, finishedWork);
        return;
      }
    case ClassComponent:
      {
        if (finishedWork.effectTag & Snapshot) {
          if (current$$1 !== null) {
            var prevProps = current$$1.memoizedProps;
            var prevState = current$$1.memoizedState;
            startPhaseTimer(finishedWork, 'getSnapshotBeforeUpdate');
            var instance = finishedWork.stateNode;
            var snapshot = instance.getSnapshotBeforeUpdate(finishedWork.elementType === finishedWork.type ? prevProps : resolveDefaultProps(finishedWork.type, prevProps), prevState);
            instance.__reactInternalSnapshotBeforeUpdate = snapshot;
            stopPhaseTimer();
          }
        }
        return;
      }
    case HostRoot:
    case HostComponent:
    case HostText:
    case HostPortal:
    case IncompleteClassComponent:
      // Nothing to do for these component types
      return;
    default:
      {
        (function () {
          {
            {
              throw ReactError('This unit of work tag should not have side-effects. This error is likely caused by a bug in React. Please file an issue.');
            }
          }
        })();
      }
  }
}
```


#### 第二次循环

第二次循环会调用 `commitAllHostEffects` 方法

主要任务是
- 重置文本节点
- 操作 ref
- 执行 插入、更新、删除的 effect 操作
- 对dom 进行操作



```
function commitAllHostEffects() {
  while (nextEffect !== null) {
    {
      setCurrentFiber(nextEffect);
    }
    recordEffect();

    var effectTag = nextEffect.effectTag;
    //重置文本节点
    if (effectTag & ContentReset) {
      commitResetTextContent(nextEffect);
    }
    //Ref有变化就detach Ref 
    if (effectTag & Ref) {
      var current$$1 = nextEffect.alternate;
      if (current$$1 !== null) {
        commitDetachRef(current$$1);
      }
    }

    // 插入，更新，删除 
    var primaryEffectTag = effectTag & (Placement | Update | Deletion);
    switch (primaryEffectTag) {
        //插入
      case Placement:
        {
          commitPlacement(nextEffect);
            //取反清除placement标志
          nextEffect.effectTag &= ~Placement;
          break;
        }
        //插入且更新
      case PlacementAndUpdate:
        {
          // Placement
          commitPlacement(nextEffect);
           //取反清除placement标志
          nextEffect.effectTag &= ~Placement;

          // Update
          var _current = nextEffect.alternate;
          commitWork(_current, nextEffect);
          break;
        }
         //更新
      case Update:
        {
          var _current2 = nextEffect.alternate;
          commitWork(_current2, nextEffect);
          break;
        }
         //删除
      case Deletion:
        {
          commitDeletion(nextEffect);
          break;
        }
    }
    执行下一个effect
    nextEffect = nextEffect.nextEffect;
  }

  {
    resetCurrentFiber();
  }
}
```


#####  commitPlacement --- 插入节点

- 根据当前的finishedWork找到tag 属于 HostComponent || HostRoot || HostPortal的parentFiber
- 获取parentFiber的dom实例，设置parent 和isContainer变量
- 由于采用insertBefore方法插入dom,需要通过getHostSibling 找到before
- 深度遍历遍历finishedWork ，有 before 进行 insertBefore，没有 before 进行 append，如果node tag 不是HostComponent || HostRoot || HostPortal ，说明其实class组件，就对其child进行操作。
  


```
function commitPlacement(finishedWork) {
  if (!supportsMutation) {
    return;
  }

  // 通过 getHostParentFiber 找到 finishedWork 的父节点 parentFiber，递归的向parent插入所有的node.
  var parentFiber = getHostParentFiber(finishedWork);

  // 这两个变量需要同时更新
  var parent = void 0;
  var isContainer = void 0;

    // 获取parent dom 节点，设置isContainer 
  switch (parentFiber.tag) {
    case HostComponent:
      parent = parentFiber.stateNode;
      isContainer = false;
      break;
    case HostRoot:
      parent = parentFiber.stateNode.containerInfo;
      isContainer = true;
      break;
    case HostPortal:
      parent = parentFiber.stateNode.containerInfo;
      isContainer = true;
      break;
    default:
      (function () {
        {
          {
            throw ReactError('Invalid host parent fiber. This error is likely caused by a bug in React. Please file an issue.');
          }
        }
      })();
  }

  if (parentFiber.effectTag & ContentReset) {
    //做任何插入操作之前重置parent的文本内容
    resetTextContent(parent);
    // 清空ContentReset 标志
    parentFiber.effectTag &= ~ContentReset;
  }

  var before = getHostSibling(finishedWork);
  
  var node = finishedWork;
  while (true) {
    if (node.tag === HostComponent || node.tag === HostText) {
      if (before) {
        if (isContainer) {
          insertInContainerBefore(parent, node.stateNode, before);
        } else {
          insertBefore(parent, node.stateNode, before);
        }
      } else {
        if (isContainer) {
          appendChildToContainer(parent, node.stateNode);
        } else {
          appendChild(parent, node.stateNode);
        }
      }
    } else if (node.tag === HostPortal) {
      // If the insertion itself is a portal, then we don't want to traverse
      // down its children. Instead, we'll get insertions from each child in
      // the portal directly.
    } else if (node.child !== null) {
        // class 组件
      node.child.return = node;
      node = node.child;
      continue;
    }
    // 当node 又执行到finishedWork时返回
    if (node === finishedWork) {
      return;
    }
    while (node.sibling === null) {
        // 兄弟节点和父节点都不存在时返回
      if (node.return === null || node.return === finishedWork) {
        return;
      }
      // 兄弟节点不存在就从父节点开始
      node = node.return;
    }
    node.sibling.return = node.return;
    node = node.sibling;
  }
}

```


getHostSibling 查找是原生类型的兄弟节点。

- 如果当前fiber不存在sibling ，判断其父节点是否存在或者父节点是否是host node，命中则返回null，否则继续向上遍历
- 如果当前fiber存在sibling,


```
function getHostSibling(fiber) {

  
  var node = fiber;
  siblings: while (true) {
    // 当前fiber 是最后一个sibling
    while (node.sibling === null) {
      if (node.return === null || isHostParent(node.return)) {
        return null;
      }
      node = node.return;
    }
    // 从兄弟节点开始找
    node.sibling.return = node.return;
    node = node.sibling;

    //如果不是原生dom 节点，继续往下遍历
    while (node.tag !== HostComponent && node.tag !== HostText && node.tag !== DehydratedSuspenseComponent) {
      if (node.effectTag & Placement) {
        continue siblings;
      }
      if (node.child === null || node.tag === HostPortal) {
        continue siblings;
      } else {
        node.child.return = node;
        node = node.child;
      }
    }
    // 是原生节点且不是新增的，返回其stateNode
    if (!(node.effectTag & Placement)) {
      // Found it!
      return node.stateNode;
    }
  }
}

```

![结构](../images/placement.jpg)

##### commitWork---更新节点

commitWork 只会更新  HostComponent(dom 节点) ， HostText(文本节点) 和 函数组件

- 更新 HostComponent时 会从当前任务 finishWork 中取出 updateQueue 、newProps、oldProps、和 dom 内容传入 commitUpdate 中从而更新到 dom 上
- 更新HostText 会调用 commitTextUpdate 更新文本内容
- 更新函数组件 会调用 commitHookEffectList，可以看到和hook有关系。


```

function commitUpdate(domElement, updatePayload, type, oldProps, newProps, internalInstanceHandle) {
  //更新fiber的internalEventHandlersKey 属性
  updateFiberProps(domElement, newProps);
  // 应用diff到dom节点
  updateProperties(domElement, updatePayload, type, oldProps, newProps);
}

function commitTextUpdate(textInstance, oldText, newText) {
  textInstance.nodeValue = newText;
}

// 处理hook的 effect

function commitHookEffectList(unmountTag, mountTag, finishedWork) {
  var updateQueue = finishedWork.updateQueue;
  var lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  if (lastEffect !== null) {
    var firstEffect = lastEffect.next;
    var effect = firstEffect;
    do {
      if ((effect.tag & unmountTag) !== NoEffect$1) {
        // Unmount
        var destroy = effect.destroy;
        effect.destroy = undefined;
        if (destroy !== undefined) {
          destroy();
        }
      }
      if ((effect.tag & mountTag) !== NoEffect$1) {
        // Mount
        var create = effect.create;
        effect.destroy = create();
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```

##### commitDeletion--- 删除节点


```
function commitDeletion(current$$1) {
  if (supportsMutation) {
      // 递归地从parent删除有所的节点，Detach refs，调用componentWillUnmount
    unmountHostComponents(current$$1);
  } else {

   // Detach refs，调用componentWillUnmount
    commitNestedUnmounts(current$$1);
  }
  // detachFiber
  detachFiber(current$$1);
}
```
##### unmountHostComponents 

```
function unmountHostComponents(current$$1) {
 
  var node = current$$1;

  var currentParentIsValid = false;

  // 这两个变量需要同事设置
  var currentParent = void 0;
  var currentParentIsContainer = void 0;



  while (true) {
    if (!currentParentIsValid) {
      var parent = node.return;
      //找到要进行删除的parent node
      findParent: while (true) {
        (function () {
          if (!(parent !== null)) {
            {
              throw ReactError('Expected to find a host parent. This error is likely caused by a bug in React. Please file an issue.');
            }
          }
        })();
        // 判断tag 是不是 HostComponent || HostRoot ||HostPortal
        switch (parent.tag) {
          case HostComponent:
            currentParent = parent.stateNode;
            currentParentIsContainer = false;
            break findParent;
          case HostRoot:
            currentParent = parent.stateNode.containerInfo;
            currentParentIsContainer = true;
            break findParent;
          case HostPortal:
            currentParent = parent.stateNode.containerInfo;
            currentParentIsContainer = true;
            break findParent;
        }
        // 不是上面三种类型就继续往上遍历
        parent = parent.return;
      }
      // 标记currentParentIsValid为true
      currentParentIsValid = true;
    }
    // 判断node 的tag 是不是 HostComponent 或者HostText
    if (node.tag === HostComponent || node.tag === HostText) {
      commitNestedUnmounts(node);
      //在所有的children卸载完毕只有，从tree上进行安全的移除
      if (currentParentIsContainer) {
        removeChildFromContainer(currentParent, node.stateNode);
      } else {
        removeChild(currentParent, node.stateNode);
      }
      // Don't visit children because we already visited them.
    } else if (enableSuspenseServerRenderer && node.tag === DehydratedSuspenseComponent) {
      // Delete the dehydrated suspense boundary and all of its content.
      if (currentParentIsContainer) {
        clearSuspenseBoundaryFromContainer(currentParent, node.stateNode);
      } else {
        clearSuspenseBoundary(currentParent, node.stateNode);
      }
    } else if (node.tag === HostPortal) {
      if (node.child !== null) {
          // 对于portal ，继续往下遍历，其子节点可能存在复合式组件
          //但是需要设置移除节点的parent为portal的containerInfo
        currentParent = node.stateNode.containerInfo;
        currentParentIsContainer = true;
        node.child.return = node;
        node = node.child;
        continue;
      }
    } else {
    //不是host component 也不是portal 
      commitUnmount(node);
    
      if (node.child !== null) {
        node.child.return = node;
        node = node.child;
        continue;
      }
    }

    // child 已经遍历完毕
    if (node === current$$1) {
      return;
    }
    // child没有了sibling ，向上遍历，直到 return 为null或者遍历到current$$1
    while (node.sibling === null) {
      if (node.return === null || node.return === current$$1) {
        return;
      }
      node = node.return;
      if (node.tag === HostPortal) {
        currentParentIsValid = false;
      }
    }
    node.sibling.return = node.return;
    node = node.sibling;
  }
}
```





#### commitNestedUnmounts

commitNestedUnmounts 遍历node进行 commitUnmount，
commitUnmount 的工作是卸载 ref， 执行 `componentWillUnmount` 生命周期

```
function commitNestedUnmounts(root) {
  
  var node = root;
  while (true) {
    commitUnmount(node);
    if (node.child !== null && (
    !supportsMutation || node.tag !== HostPortal)) {
      node.child.return = node;
      node = node.child;
      continue;
    }
    // 接下来向上遍历
    if (node === root) {
      return;
    }
    while (node.sibling === null) {
      if (node.return === null || node.return === root) {
        return;
      }
      node = node.return;
    }
    node.sibling.return = node.return;
    node = node.sibling;
  }
}
```


#### 第三次循环

这个阶段主要是执行生命周期函数

- 执行componentDidMount
- 执行 componentDidUpdate
- 执行 setState 的 callback 回调函数，准确点说应该是update 的callback


commitLifeCycles

```
function commitAllLifeCycles(finishedRoot, committedExpirationTime) {
  {
    ReactStrictModeWarnings.flushPendingUnsafeLifecycleWarnings();
    ReactStrictModeWarnings.flushLegacyContextWarning();

    if (warnAboutDeprecatedLifecycles) {
      ReactStrictModeWarnings.flushPendingDeprecationWarnings();
    }
  }
  while (nextEffect !== null) {
    {
      setCurrentFiber(nextEffect);
    }
    var effectTag = nextEffect.effectTag;

    if (effectTag & (Update | Callback)) {
      recordEffect();
      var current$$1 = nextEffect.alternate;
      commitLifeCycles(finishedRoot, current$$1, nextEffect, committedExpirationTime);
    }

    if (effectTag & Ref) {
      recordEffect();
      commitAttachRef(nextEffect);
    }

    if (effectTag & Passive) {
      rootWithPendingPassiveEffects = finishedRoot;
    }

    nextEffect = nextEffect.nextEffect;
  }
  {
    resetCurrentFiber();
  }
}
```

commitLifeCycles 根据不同的tab进行对应的处理。

- 如果是 FunctionComponent||ForwardRef||SimpleMemoComponent会再次调用commitHookEffectList，可以看到其参数在不同阶段是不一样的。本次循环会传入 UnmountLayout, MountLayout。
- 如果是ClassComponent ，会判断是首次渲染还是更新渲染决定是调用componentDidMount还是componentDidUpdate
- 如果当前fiber的updateQueue 存在会调用commitUpdateQueue
- 如果是HostRoot，调用commitUpdateQueue
- 如果是HostComponent ，调用commitMount


```
function commitLifeCycles(finishedRoot, current$$1, finishedWork, committedExpirationTime) {
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case SimpleMemoComponent:
      {
        commitHookEffectList(UnmountLayout, MountLayout, finishedWork);
        break;
      }
    case ClassComponent:
      {
        var instance = finishedWork.stateNode;
        if (finishedWork.effectTag & Update) {
          if (current$$1 === null) {
            startPhaseTimer(finishedWork, 'componentDidMount');
            instance.componentDidMount();
            stopPhaseTimer();
          } else {
            var prevProps = finishedWork.elementType === finishedWork.type ? current$$1.memoizedProps : resolveDefaultProps(finishedWork.type, current$$1.memoizedProps);
            var prevState = current$$1.memoizedState;
            startPhaseTimer(finishedWork, 'componentDidUpdate');
            instance.componentDidUpdate(prevProps, prevState, instance.__reactInternalSnapshotBeforeUpdate);
            stopPhaseTimer();
          }
        }
        var updateQueue = finishedWork.updateQueue;
        if (updateQueue !== null) {
          
          commitUpdateQueue(finishedWork, updateQueue, instance, committedExpirationTime);
        }
        return;
      }
    case HostRoot:
      {
        var _updateQueue = finishedWork.updateQueue;
        if (_updateQueue !== null) {
          var _instance = null;
          if (finishedWork.child !== null) {
            switch (finishedWork.child.tag) {
              case HostComponent:
                _instance = getPublicInstance(finishedWork.child.stateNode);
                break;
              case ClassComponent:
                _instance = finishedWork.child.stateNode;
                break;
            }
          }
          commitUpdateQueue(finishedWork, _updateQueue, _instance, committedExpirationTime);
        }
        return;
      }
    case HostComponent:
      {
        var _instance2 = finishedWork.stateNode;

        if (current$$1 === null && finishedWork.effectTag & Update) {
          var type = finishedWork.type;
          var props = finishedWork.memoizedProps;
          commitMount(_instance2, type, props, finishedWork);
        }

        return;
      }
    case HostText:
      {
        // We have no life-cycles associated with text.
        return;
      }
    case HostPortal:
      {
        // We have no life-cycles associated with portals.
        return;
      }
    case Profiler:
      {
        if (enableProfilerTimer) {
          var onRender = finishedWork.memoizedProps.onRender;

          if (enableSchedulerTracing) {
            onRender(finishedWork.memoizedProps.id, current$$1 === null ? 'mount' : 'update', finishedWork.actualDuration, finishedWork.treeBaseDuration, finishedWork.actualStartTime, getCommitTime(), finishedRoot.memoizedInteractions);
          } else {
            onRender(finishedWork.memoizedProps.id, current$$1 === null ? 'mount' : 'update', finishedWork.actualDuration, finishedWork.treeBaseDuration, finishedWork.actualStartTime, getCommitTime());
          }
        }
        return;
      }
    case SuspenseComponent:
    case IncompleteClassComponent:
      break;
    default:
      {
        (function () {
          {
            {
              throw ReactError('This unit of work tag should not have side-effects. This error is likely caused by a bug in React. Please file an issue.');
            }
          }
        })();
      }
  }
}
```
###### commitUpdateQueue

会找到此次更新 setState 的回调进行执行，当更新中有捕获错误的回调函数也会在这个时机执行，目的是将错误在componentDidCatch进行捕获. 


```
function commitUpdateQueue(finishedWork, finishedQueue, instance, renderExpirationTime) {
  
  // 如果有捕获的错误，因其优先级低，需要将其保存在queue中以便处理完其他任务后进行处理
  if (finishedQueue.firstCapturedUpdate !== null) {
    // Join the captured update list to the end of the normal list.
    if (finishedQueue.lastUpdate !== null) {
      finishedQueue.lastUpdate.next = finishedQueue.firstCapturedUpdate;
      finishedQueue.lastUpdate = finishedQueue.lastCapturedUpdate;
    }
    // Clear the list of captured updates.
    finishedQueue.firstCapturedUpdate = finishedQueue.lastCapturedUpdate = null;
  }

  // Commit the effects
  commitUpdateEffects(finishedQueue.firstEffect, instance);
  finishedQueue.firstEffect = finishedQueue.lastEffect = null;
  // 异常处理update队列
  commitUpdateEffects(finishedQueue.firstCapturedEffect, instance);
  finishedQueue.firstCapturedEffect = finishedQueue.lastCapturedEffect = null;
}

// 取出effect的callback，对于hostRoot节点来说callback是 ReactWork.prototype._onCommit 可参考`scheduleRootUpdate`方法
//callback 一般是setState的回调函数
function commitUpdateEffects(effect, instance) {
  while (effect !== null) {
    var _callback3 = effect.callback;
    if (_callback3 !== null) {
      effect.callback = null;
      callCallback(_callback3, instance);
    }
    effect = effect.nextEffect;
  }
}
```

### 收尾工作

重置变量和其他工作
```
  isCommitting$1 = false;
  isWorking = false;
  stopCommitLifeCyclesTimer();
  stopCommitTimer();

  root.expirationTime = expirationTime;
  root.finishedWork = null;
```
