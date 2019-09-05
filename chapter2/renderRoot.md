<!--
 * @Description: In User Settings Edit
 * @Author: your name
 * @Date: 2019-07-28 00:28:22
 * @LastEditTime: 2019-09-05 14:55:01
 * @LastEditors: Please set LastEditors
 -->
## renderRoot

renderRoot会先判断是否是一个从新开始的root， 是的话会重置各个属性

首先是resetStack()函数， 对原有的进行中的root任务中断， 进行存储，
紧接着将nextRoot和nextRendeExpirationTime重置， 同时创建第一个nextUnitOfWork, 也就是一个工作单元
这个nextUnitOfWork也是一个workProgress, 也是root.current的alternater属性， 而它的alternate属性则指向了root.current， 形成了一个双缓冲池。

```
if (expirationTime !== nextRenderExpirationTime || root !== nextRoot || nextUnitOfWork === null) {
    // Reset the stack and start working from the root.
    resetStack();
    nextRoot = root;
    nextRenderExpirationTime = expirationTime;
    nextUnitOfWork = createWorkInProgress(nextRoot.current, null, nextRenderExpirationTime);
    root.pendingCommitExpirationTime = NoWork;
}
```

接着执行wookLoop（isYield）函数， 该函数通过循环执行， 遍历每一个nextUniOfWork

```
function workLoop(isYieldy) {
  if (!isYieldy) {
    // Flush work without yielding
    while (nextUnitOfWork !== null) {
      nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    }
  } else {
    // Flush asynchronous work until there's a higher priority event
    while (nextUnitOfWork !== null && !shouldYieldToRenderer()) {
      nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    }
  }
}
```

## performUnitOfWork

```
function performUnitOfWork(workInProgress) {
  var current$$1 = workInProgress.alternate;
  var next = void 0;
  next = beginWork(current$$1, workInProgress, nextRenderExpirationTime);
  workInProgress.memoizedProps = workInProgress.pendingProps;

  if (next === null) {
    // 点前fiber已经处理到叶子节点了，完成当前节点，并找到下一个要处理的节点
    next = completeUnitOfWork(workInProgress);
  }

  ReactCurrentOwner$2.current = null;

  return next;
}
```

performUnitOfWork 先 获取 参数的alaernate属性， 赋值给current

beginWork主要根据workInprogress的tag来做不同的处理,进行节点操作，以及创建子节点，子节点会返回成为next，如果有next就返回到workLoop，nextUnitOfWork不为null时会再次调用performUnitOfWork。

如果next不存在，说明当前节点向下遍历子节点已经到底了，说明这个子树侧枝已经遍历完，可以完成这部分工作了，就执行completeUnitOfWork方法

completeUnitOfWork 主要是处理effect,包括标记effect tag,向上收敛effect list。它会判断当前节点是否有sibiling, 有则直接返回赋值给next, 否则判断父fiber是否有sibiling, 一直循环到最上层 父fiber为null， 执行的同时会把effect逐级传给父fiber。到了root节点，表明整棵树的遍历已经结束了，可以commit了，然后循环结束，返回到renderRoot。



 
 循环结束之后调用onComplete

 ```
 function onComplete(root, finishedWork, expirationTime) {
  root.pendingCommitExpirationTime = expirationTime;
  root.finishedWork = finishedWork;
}
 ```

返回到performWorkOnRoot函数， 进入commit阶段， 将isRendering状态设为false,返回到performWork函数， 继续进入循环执行root， 直到所有root完成


以上是整体的粗略流程。



## beginWork

####  beginWork有三个核心功能


- 验证当前 fiber 树是否需要更新，用于性能优化
- 根据传入的节点类型进行对应的更新
- 更新后调和子节点

用于性能优化的判断
```
var updateExpirationTime = workInProgress.expirationTime;
if(updateExpirationTime < renderExpirationTime) {
     return bailoutOnAlreadyFinishedWork(current$$1, workInProgress, renderExpirationTime);
}

```
如果expirationTime小于当前渲染的超时时间，说明这个fiber没有任何需要处理的任务，直接调用``` bailoutOnAlreadyFinishedWork ```


```
function bailoutOnAlreadyFinishedWork(current$$1, workInProgress, renderExpirationTime) {
  cancelWorkTimer(workInProgress);

  if (current$$1 !== null) {
    // Reuse previous context list
    workInProgress.contextDependencies = current$$1.contextDependencies;
  }

  // Check if the children have any pending work.
  var childExpirationTime = workInProgress.childExpirationTime;
  if (childExpirationTime < renderExpirationTime) {
    // The children don't have any work either. We can skip them.
    // TODO: Once we add back resuming, we should check if the children are
    // a work-in-progress set. If so, we need to transfer their effects.
    return null;
  } else {
    // This fiber doesn't have work, but its subtree does. Clone the child
    // fibers and continue.
    cloneChildFibers(current$$1, workInProgress);
    return workInProgress.child;
  }
}
```
bailoutOnAlreadyFinishedWork会检查```childExpirationTime < renderExpirationTime```
- 如果子树也没有更新，就直接返回null，跳过子树的reconcilation.
- 如果不满足表明自子节点有工作要进行，就复用当前Fiber对象，然后返回他的子节点赋值workInProgress.child。


#####  根据 ```workInProgress.tag```执行不同的方法，进行更新。

进行更新先把当前 fiber 的 expirationTime 设置为 NoWork，根据 tag 进行不同组件的更新
```
workInProgress.expirationTime = NoWork;
更新方法如下：
switch (workInProgress.tag) {
    case:mountIndeterminateComponent:....
    case:mountLazyComponent:....
    case:updateFunctionComponent:....
    case:updateClassComponent:....
    case:updateHostRoot:....
    case:updateHostComponent:....
    case:updateHostText:....
    case:updateSuspenseComponent:....
    case:updatePortalComponent:....
    case:updateForwardRef:....
    case:updateFragment:....
    case:updateMode:....
    case:updateProfiler:....
    case:updateContextProvider:....
    case:updateContextConsumer:....
    case:updateMemoComponent:....
    case:updateSimpleMemoComponent:....
    case:mountIncompleteClassComponent:....
    case:updateDehydratedSuspenseComponent:....
}
```
可以看出beginWork主要目的是在当前节点进行工作，工作内容包括实例化class，更新state,props，等等，然后返回组件的子节点```workInProgress.child```来进行后续的工作。

这些方法后面会详细介绍。

当当前节点是叶子节点的时候，workInProgress.child=null ,会进入 `completeUnitOfWork`


