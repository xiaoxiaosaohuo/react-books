## scheduleWork
scheduleWork先执行一个scheduleWorkToRoot函数， 该函数主要是更新其expirationTime以及上层fiber的childrenExpirationTime

```
function scheduleWork(fiber, expirationTime) {
  var root = scheduleWorkToRoot(fiber, expirationTime);
  if (root === null) {
    {
      switch (fiber.tag) {
        case ClassComponent:
          warnAboutUpdateOnUnmounted(fiber, true);
          break;
        case FunctionComponent:
        case ForwardRef:
        case MemoComponent:
        case SimpleMemoComponent:
          warnAboutUpdateOnUnmounted(fiber, false);
          break;
      }
    }
    return;
  }

  if (!isWorking && nextRenderExpirationTime !== NoWork && expirationTime > nextRenderExpirationTime) {
    // This is an interruption. (Used for performance tracking.)
    interruptedBy = fiber;
    resetStack();
  }
  markPendingPriorityLevel(root, expirationTime);
  if (
  // If we're in the render phase, we don't need to schedule this root
  // for an update, because we'll do it before we exit...
  !isWorking || isCommitting$1 ||
  // ...unless this is a different root than the one we're rendering.
  nextRoot !== root) {
    var rootExpirationTime = root.expirationTime;
    requestWork(root, rootExpirationTime);
  }
  if (nestedUpdateCount > NESTED_UPDATE_LIMIT) {
    // Reset this back to zero so subsequent updates don't throw.
    nestedUpdateCount = 0;
    invariant(false, 'Maximum update depth exceeded. This can happen when a component repeatedly calls setState inside componentWillUpdate or componentDidUpdate. React limits the number of nested updates to prevent infinite loops.');
  }
}
```


## scheduleWorkToRoot

- 更新当前fiber的expirationTime和其alternate的expirationTime
- 向上遍历一直到root,更新每个节点的childExpirationTime
- 返回 root 节点的Fiber对象

```
function scheduleWorkToRoot(fiber, expirationTime) {
  recordScheduleUpdate();

  {
    if (fiber.tag === ClassComponent) {
      var instance = fiber.stateNode;
      warnAboutInvalidUpdates(instance);
    }
  }

  // Update the source fiber's expiration time
  if (fiber.expirationTime < expirationTime) {
    fiber.expirationTime = expirationTime;
  }
  var alternate = fiber.alternate;
  if (alternate !== null && alternate.expirationTime < expirationTime) {
    alternate.expirationTime = expirationTime;
  }
  // Walk the parent path to the root and update the child expiration time.
  var node = fiber.return;
  var root = null;
  if (node === null && fiber.tag === HostRoot) {
    root = fiber.stateNode;
  } else {
    // 循环查找 FiberRoot
    while (node !== null) {
      alternate = node.alternate;
      if (node.childExpirationTime < expirationTime) {
        node.childExpirationTime = expirationTime;
        if (alternate !== null && alternate.childExpirationTime < expirationTime) {
          alternate.childExpirationTime = expirationTime;
        }
      } else if (alternate !== null && alternate.childExpirationTime < expirationTime) {
        alternate.childExpirationTime = expirationTime;
      }
      // 找到 FiberRoot 结束循环
      if (node.return === null && node.tag === HostRoot) {
        root = node.stateNode;
        break;
      }
      // 继续向父节点查找
      node = node.return;
    }
  }
  return root;
}
```

然后要进行一个判断：
```
if (!isWorking && nextRenderExpirationTime !== NoWork && expirationTime > nextRenderExpirationTime) {
    // This is an interruption. (Used for performance tracking.)
    interruptedBy = fiber;
    resetStack();
  }
```

- isWorking代表是否正在工作，在开始renderRoot和commitRoot的时候会设置为 true
- nextRenderExpirationTime在是新的renderRoot的时候会被设置为当前任务的expirationTime
  
整个判断的意思就是：*** 目前没有任何任务在执行，并且之前有执行过任务，同时当前的任务比之前执行的任务过期时间大（也就是优先级要高） ***

那么这种情况会出现在什么时候呢？

答案就是：*** 上一个任务是异步任务，优先级很低，并且在上一个时间片（初始是 33ms）任务没有执行完，而且等待下一次requestIdleCallback的时候新的任务进来了，并且超时时间很短，那么优先级就变成了先执行当前任务，正如注释中所示这是一次打断 ，任务打断之后，之前任务会从stack中pop出去 ***


#### resetStack

- 当发现当前任务的优先级大于下一个任务的优先级时，把下个任务的优先级重置执行当前任务
- resetStack 在重置下个任务时，会先记录这个任务，等待以后执行，并且使用 unwindInterruptedWork 来重置这个任务 fiber 上级的状态

而且resetStack方法会重置一下变量
```
  nextRoot = null;
  nextRenderExpirationTime = NoWork;
  nextLatestAbsoluteTimeoutMs = -1;
  nextRenderDidError = false;
  nextUnitOfWork = null;
```

## markPendingPriorityLevel
这个函数主要是更新root的最高优先级和最低优先级（earliestPendingTime和lastestPendingTime;）, 同时设置下一个执行操作的时间nextExpirationTimeToWorkOn（即root中具有最高优先级的fiber的expirationTime）

## requestWork

```
// requestWork is called by the scheduler whenever a root receives an update.
// It's up to the renderer to call renderRoot at some point in the future.
function requestWork(root, expirationTime) {
  addRootToSchedule(root, expirationTime);
  if (isRendering) {
   //防止重新进入，剩余的工作将安排在当前渲染批次最后进行
    return;
  }
  // 批量处理相关 调用 setState 时在 enqueueUpdates 前 batchedUpdates 会把 isBatchingUpdates 设置成 true
  if (isBatchingUpdates) {
    // Flush work at the end of the batch.
    if (isUnbatchingUpdates) {
      // ...unless we're inside unbatchedUpdates, in which case we should
      // flush it now.
      nextFlushedRoot = root;
      nextFlushedExpirationTime = Sync;
      performWorkOnRoot(root, Sync, false);
    }
    return;
  }

  if (expirationTime === Sync) {
    performSyncWork();
  } else {
    // 异步调度 利用浏览器有空闲的时候进行执行
    scheduleCallbackWithExpirationTime(root, expirationTime);
  }
}

function addRootToSchedule(root: FiberRoot, expirationTime: ExpirationTime) {
  // root.nextScheduledRoot 用来判断是否有异步任务正在调度, 为 null 时会增加 nextScheduledRoot
  // 这个 root 还没有进入过调度
  if (root.nextScheduledRoot === null) {
    root.expirationTime = expirationTime;
    // lastScheduledRoot firstScheduledRoot 是单向链表结构，表示多个 root 更新
    // 这里只有一个 root 只会在这里执行
    if (lastScheduledRoot === null) {
      firstScheduledRoot = lastScheduledRoot = root;
      root.nextScheduledRoot = root;
    } else { // 有个多个root 时进行单向链表的插入操作
      lastScheduledRoot.nextScheduledRoot = root;
      lastScheduledRoot = root;
      lastScheduledRoot.nextScheduledRoot = firstScheduledRoot;
    }
  } else { 
    // 传入的 root 已经进入过调度, 把 root 的优先级设置最高
    const remainingExpirationTime = root.expirationTime;
    // 如果 root 的 expirationTime 是同步或者优先级低，设置为计算出的最高优先级
    if (
      remainingExpirationTime === NoWork ||
      expirationTime < remainingExpirationTime
    ) {
      root.expirationTime = expirationTime; // 把当前 root 的优先级设置为当前优先级最高的
    }
  }
}
```
1. addRootToSchedule把 root 加入到调度队列,第一次调用addRootToSchedule的时候，nextScheduledRoot是null，这时候公共变量firstScheduledRoot和lastScheduledRoot也是null，所以会把他们都赋值成root，同时root.nextScheduledRoot = root。然后第二次进来的时候，如果前后root是同一个，那么之前的firstScheduledRoot和lastScheduledRoot都是 root，所以lastScheduledRoot.nextScheduledRoot = root就等于root.nextScheduledRoot = root

这么做是因为同一个root不需要存在两个，因为前一次调度如果中途被打断，下一次调度进入还是从同一个root开始，就会把新的任务一起执行了。

2. 同时更新root的expirationTime属性
3. isRendering 正在rendering就返回，剩余的工作将安排在当前渲染批次最后进行
4. isBatchingUpdates和isUnbatchingUpdates涉及到事件系统，触发react合成事件时isBatchingUpdates会置为true,此处react的state更新是会合并的。
5. 根据expirationTime调用performSyncWork还是scheduleCallbackWithExpirationTime
6. scheduleCallbackWithExpirationTime是根据时间片来执行任务的，会涉及到requestIdleCallback


先看同步方法 ``` performWork ```


