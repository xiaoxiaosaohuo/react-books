## Fiber工作流程

主要有两个阶段
1. reconciliation/render  可暂停，恢复等。
2. commit 一次性完成


### reconciliation/render
render的重点是确定需要插入，更新或删除哪些节点，以及哪些组件需要调用其生命周期方法。

在reconciliation 时，自顶向下逐节点构造workInProgress tree。
会根据render生成的react elements生成对应的fiber nodes tree, 每一个element 对应一个fiber node，fiber持有组件的状态和DOM，用于描述需要完成的工作。

react element在每次render的时候都会新建，但是fiber并不会，如果current.alertnate存在则会重用之前的fiber，仅仅把相应的react element的属性更新到这个fiber上 见```createWorkInProgress ```方法。

创建fiber时根据不同type会调用不同的方法进行的处理，但是最后都会调用creteFiber方法来创建fiber
```
createHostRootFiber
createFiberFromElement
createFiberFromFragment
createFiberFromPortal
createFiberFromText
...

fiber = createFiber(fiberTag, pendingProps, key, mode)
```

这是一个链表结构的fiber tree.

各节点之间通过 return，child，sibling属性进行关联

- return 父级节点
- child 子节点
- sibling 兄弟节点

![fibertree](https://user-images.githubusercontent.com/15315816/50039685-0b283680-0071-11e9-9d13-681b697719d2.png)


#### Fiber Reconciliation流程分析

1. 如果当前节点不需要更新，则把子节点clone过来，跳到步骤5
2. 跟新当前节点的props,state,context等，调用getDerivedStateFromProps静态方法
3. 调用shouldComponentUpdate()，false的话，跳到步骤5
4. 调用组件的render方法获得新的children, 进行reconcileChildren，为子节点创建fiber（创建过程会尽量复用现有fiber，子节点增删移动发生在这里，不是真实的dom节点，并标记effectTag，收集effect）
5. 如果没有workInProgress.child，则工作单元结束，调用completeWork，如果是dom节点就diffProperties,标记tag，把effect list归并到return fiber，并把当前节点的sibling作为下一个工作单元；否则把child作为下一个工作单元
6. 如果没有剩余可用时间了，等到下一次主线程空闲时才开始下一个工作单元；否则，立即开始做
7. 当返回到root节点的时候，工作循环结束，进入commit阶段
8. commit阶段，标记isWorking = true;此阶段不可打断
9. 有三个大循环遍历effect list。主要是调用生命周期，将更新flush到屏幕上

#### 工作循环和主要方法介绍

```
function workLoop(isYieldy) {
    if (!isYieldy) {
        // Flush work without yielding
        while (nextUnitOfWork !== null) {
            nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
        }
    } else {
        // Flush asynchronous work until the deadline runs out of time.
        while (nextUnitOfWork !== null && !shouldYield()) {
            nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
        }
    }
}

function performUnitOfWork(workInProgress){
  var current$$1 = workInProgress.alternate;
  var next = void 0;
  //始终返回指向要在循环中处理的下一个子节点的指针或null。
  next = beginWork(current$$1, workInProgress, nextRenderExpirationTime);
  if (next === null) {
    // If this doesn't spawn new work, complete the current work.
    next = completeUnitOfWork(workInProgress);
  }
  return next;
} 

function completeUnitOfWork(workInProgress) {
    while (true) {
        let returnFiber = workInProgress.return;
        let siblingFiber = workInProgress.sibling;

        nextUnitOfWork = completeWork(workInProgress);

        if (siblingFiber !== null) {
            // If there is a sibling, return it
            // to perform work for this sibling
            return siblingFiber;
        } else if (returnFiber !== null) {
            // If there's no more work in this returnFiber,
            // continue the loop to complete the parent.
            workInProgress = returnFiber;
            continue;
        } else {
            // We've reached the root.
            return null;
        }
    }
}


```

nextUnitOfWork 指向workInProgress的fiber node,随着遍历的进行，使用该变量能判断是否有未完成的工作，在current fiber处理完之后，nextUnitOfWork指向下一个fiber node或null,当返回null时，会退出workLoop,准备进入commit 阶段。

#### Side-effects

state和props的改变会产生 side-effects，每个fiber node都有与之关联的effect,用effectTag 进行标识，其表示在处理更新时需要在实例上完成的工作

对于host components来说，可以有增加，删除，更新。对于class component来说，要更新其refs,调用componentDidMount，componentDidUpdate等生命周期。

对于hooks来说就是useEffect中的方法调用。


React 使用单链表存储effect, 通过nextEffect指向下一个effect


![fibertree1](https://user-images.githubusercontent.com/15315816/50039691-22672400-0071-11e9-83f0-a1410ba9eb09.png)


### Commit阶段


这是React调用生命周期并更新DOM的地方

这个阶段，有两颗树，current和workInProgress

- current代表当前屏幕上呈现的状态
- workInProgress或者finishedWork代表需要刷新到屏幕上的状态

这个阶段的主要工作如下

1. 调用getSnapshotBeforeUpdate生命周期钩子
2. 在effectTag=Deletion的节点上调用componentWillUnmount生命周期方法
3. 设置 finishedWork tree 为 current
4. 调用 componentDidMount生命周期 如果该节点标记为 Placement
5. 调用 componentDidUpdate生命周期 如果该节点标记为 Update

简化流程如下

```
function commitRoot(root, finishedWork) {
    commitBeforeMutationLifecycles()
    commitAllHostEffects();
    root.current = finishedWork;
    commitAllLifeCycles();
}
```


这个commit 阶段核心是三个大循环，遍历effects list
**第一个循环** commitBeforeMutationLifecycles 调用组件的getSnapshotBeforeUpdate 
```
function commitBeforeMutationLifecycles() {
  while (nextEffect !== null) {

    var effectTag = nextEffect.effectTag;
    if (effectTag & Snapshot) {
      recordEffect();
      var current$$1 = nextEffect.alternate;
      commitBeforeMutationLifeCycles(current$$1, nextEffect);
    }

    nextEffect = nextEffect.nextEffect;
  }
}
```
**第二个循环** commitAllHostEffects ，根据effectTag应用更新，有三种tag,(Placement | Update | Deletion)
```
function commitAllHostEffects() {
  while (nextEffect !== null) {

    var effectTag = nextEffect.effectTag;

    if (effectTag & ContentReset) {
      commitResetTextContent(nextEffect);
    }

    if (effectTag & Ref) {
      var current$$1 = nextEffect.alternate;
      if (current$$1 !== null) {
        commitDetachRef(current$$1);
      }
    }

    var primaryEffectTag = effectTag & (Placement | Update | Deletion);
    switch (primaryEffectTag) {
      case Placement:
        {
          commitPlacement(nextEffect);
          nextEffect.effectTag &= ~Placement;
          break;
        }
      case PlacementAndUpdate:
        {
          // Placement
          commitPlacement(nextEffect);
          nextEffect.effectTag &= ~Placement;

          // Update
          var _current = nextEffect.alternate;
          commitWork(_current, nextEffect);
          break;
        }
      case Update:
        {
          var _current2 = nextEffect.alternate;
          commitWork(_current2, nextEffect);
          break;
        }
      case Deletion:
        {
          commitDeletion(nextEffect);
          break;
        }
    }
    nextEffect = nextEffect.nextEffect;
  }

}
```




**第三个循环**  commitAllLifeCycles 调用生命周期，初次挂载调用componentDidMount，更新阶段调用componentDidUpdate 


```
function commitAllLifeCycles(finishedRoot, committedExpirationTime) {
  while (nextEffect !== null) {
    var effectTag = nextEffect.effectTag;

    if (effectTag & (Update | Callback)) {
      var current$$1 = nextEffect.alternate;
      commitLifeCycles(finishedRoot, current$$1, nextEffect, committedExpirationTime);
    }

    if (effectTag & Ref) {
      commitAttachRef(nextEffect);
    }

    if (effectTag & Passive) {
      rootWithPendingPassiveEffects = finishedRoot;
    }

    nextEffect = nextEffect.nextEffect;
  }
}
```




