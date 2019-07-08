```
root = ({
      current: uninitializedFiber,
        // 代表当前对应的fiber，这里是未初始化的fiber
      containerInfo: containerInfo,
        // 代表容器的节点
      pendingChildren: null,
        //只有在持久化更新的平台会用到，在react-Dom中不会被用到
      earliestPendingTime: NoWork,
         //最老的正在进行中的任务，这里初始化都为Nowork为0，最低优先级
      latestPendingTime: NoWork,
          //最新的正在进行中的任务
      earliestSuspendedTime: NoWork,
          //最老的被挂起的任务
      latestSuspendedTime: NoWork,
          //最新的被挂起的任务
      latestPingedTime: NoWork,
            
      pingCache: null,

      didError: false,
       //标记整个应用在渲染的过程中是否有错误
      pendingCommitExpirationTime: NoWork,
      //正在提交的任务的ExpirationTime，也就是优先级
      finishedWork: null,
      //在render阶段已经完成了的任务，在commit阶段只会执行finishedWork的任务
      timeoutHandle: noTimeout,
      //用来清理还没有被触发的计时器
      context: null,
      //顶层的context对象，只用在调用“renderSubTreeIntoContainer”的时候在有用
      pendingContext: null,
      hydrate,
      //应用是否要和原来的dom节点进行合并
      nextExpirationTimeToWorkOn: NoWork,
      //记录下一次将要进行的对应的优先级的任务
      expirationTime: NoWork,
      //当前的优先级的任务
      firstBatch: null,
      
      nextScheduledRoot: null,
        //链表的结构，两次react-Domrender渲染的。。节点的链表
      interactionThreadID: unstable_getThreadID(),
      // 交互的线程id
      memoizedInteractions: new Set(),
      //上次交互的线程id的Set对象
      pendingInteractionMap: new Map(),
      //进行中的交互的线程的Map对象
    }: FiberRoot);
```

## expirationTime计算

最终的公式 ```((((currentTime - 2 + 5000 / 10) / 25) | 0) + 1) * 25```, 公式用了 | 0取整，使得在 * 25 倍数之间很短的一段时间粒度里计算出的值是一样的
在很短时间内 setState 多次调用更新时，也可以保证是同一优先级的更新。

