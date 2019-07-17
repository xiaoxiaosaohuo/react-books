## renderRoot

renderRoot会先判断是否是一个从新开始的root， 是的话会重置各个属性

首先是resetStack()函数， 对原有的进行中的root任务中断， 进行存储
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
    // If this doesn't spawn new work, complete the current work.
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



