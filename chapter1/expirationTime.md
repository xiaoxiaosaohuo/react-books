## expirationTime
过期时间，与任务单元的优先级相关，根据expirationTime来判断是否进行下一个分片任务，过期时间决定了更新的批处理方式。

***expirationTime越大优先级越高***


```
// Max 31 bit integer. The max integer size in V8 for 32-bit systems.
// Math.pow(2, 30) - 1
// 0b111111111111111111111111111111
var maxSigned31BitInt = 1073741823;

var NoWork = 0;// 没有任务等待处理
var Never = 1;// 暂不执行，优先级最低
var Sync = maxSigned31BitInt;// 同步模式，立即处理，优先级最高

var UNIT_SIZE = 10; // 过期时间单元（ms）
var MAGIC_NUMBER_OFFSET = maxSigned31BitInt - 1; // 到期时间偏移量
```
maxSigned31BitInt，通过注释可以知道这是32位系统V8引擎里最大的整数。据粗略计算这个时间大概是12.427天。

####  expirationTime计算公式

```

// 1个单元的过期时间是10ms.
function msToExpirationTime(ms) {
  // Always add an offset so that we don't clash with the magic number for NoWork.
  return MAGIC_NUMBER_OFFSET - (ms / UNIT_SIZE | 0);
}

function expirationTimeToMs(expirationTime) {
  return (MAGIC_NUMBER_OFFSET - expirationTime) * UNIT_SIZE;
}

function ceiling(num, precision) {
  return ((num / precision | 0) + 1) * precision;
}

function computeExpirationBucket(currentTime, expirationInMs, bucketSizeMs) {
  return MAGIC_NUMBER_OFFSET - ceiling(MAGIC_NUMBER_OFFSET - currentTime + expirationInMs / UNIT_SIZE, bucketSizeMs / UNIT_SIZE);
}

var LOW_PRIORITY_EXPIRATION = 5000;
var LOW_PRIORITY_BATCH_SIZE = 250;

function computeAsyncExpiration(currentTime) {
  return computeExpirationBucket(currentTime, LOW_PRIORITY_EXPIRATION, LOW_PRIORITY_BATCH_SIZE);
}
var HIGH_PRIORITY_EXPIRATION = __DEV__ ? 500 : 150;
var HIGH_PRIORITY_BATCH_SIZE = 100;

function computeInteractiveExpiration(currentTime) {
  return computeExpirationBucket(currentTime, HIGH_PRIORITY_EXPIRATION, HIGH_PRIORITY_BATCH_SIZE);
}
```

> 上面主要有两个方法，可以看到React把到期时间主要分为两种：1.异步任务到期时间和交互动作到期时间。Interactive的比如说是由事件触发的，那么它的响应优先级会比较高因为涉及到交互。

## ceiling
方法的作用是向上取整，```|0```表示向下取整，再加1，即向上取整。间隔在precision内的两个num最终得到的相同的值。 如果precision为25，则num为50和70转换后的到期时间都是75。这样相差25ms内的当前时间经过计算被统一为同样的过期时间，让非常相近的两次更新得到相同的expirationTime，然后在一次更新中完成，相当于一个自动的batchedUpdates，减少渲染次数。

## computeExpirationBucket
参数：
- currentTime 需要转换的当前时间
- expirationInMs 不同优先级的异步任务对应的偏移时间
- bucketSizeMs 精确度

如果是低优先级的异步任务，则第二个参数expirationInMs传入LOW_PRIORITY_EXPIRATION = 5000。LOW_PRIORITY_BATCH_SIZE = 250; 会抹平25ms内的差异

> 第二个参数expirationInMs 的作用就是如果在一个当前时间左右的不同优先级任务的到期时间相差无几的话，不加上这个参数，就无法区分优先级，计算的过期时间可能是一样的。

```
function computeExpirationBucket(currentTime, expirationInMs, bucketSizeMs) {
  return MAGIC_NUMBER_OFFSET - ceiling(MAGIC_NUMBER_OFFSET - currentTime + expirationInMs / UNIT_SIZE, bucketSizeMs / UNIT_SIZE);
}
```
computeAsyncExpiration计算出来的优先级低于computeInteractiveExpiration的，因为computeInteractiveExpiration涉及到交互。


## computeAsyncExpiration


```
var LOW_PRIORITY_EXPIRATION = 5000;
var LOW_PRIORITY_BATCH_SIZE = 250;

function computeAsyncExpiration(currentTime) {
  return computeExpirationBucket(currentTime, LOW_PRIORITY_EXPIRATION, LOW_PRIORITY_BATCH_SIZE);
}

此时 bucketSizeMs / UNIT_SIZE  =  250/10 = 25ms


```

当currentTime为50和71，经过转换后到期时间都是 1073742275

```
50: ((1073741822-50+500)/25|0+1)*25 = 1073742275
71: ((1073741822-71+500)/25|0+1)*25 = 1073742275
```


## computeInteractiveExpiration

```
var HIGH_PRIORITY_EXPIRATION = __DEV__ ? 500 : 150;
var HIGH_PRIORITY_BATCH_SIZE = 100;

function computeInteractiveExpiration(currentTime) {
  return computeExpirationBucket(currentTime, HIGH_PRIORITY_EXPIRATION, HIGH_PRIORITY_BATCH_SIZE);
}

```
交互任务在开发环境得到的到期时间大于生产环境


## 获取当前时间 currentTime

requestCurrentTime方法用于获取当前时间

```
function requestCurrentTime() {
      if (isRendering) {
        return currentSchedulerTime;
      }
      findHighestPriorityRoot();
      if (
        nextFlushedExpirationTime === NoWork ||
        nextFlushedExpirationTime === Never
      ) {
        recomputeCurrentRendererTime();
        currentSchedulerTime = currentRendererTime;
        return currentSchedulerTime;
      }
      return currentSchedulerTime;
}

```

在 React 中我们计算expirationTime要基于当前得时钟时间，一般来说我们只需要获取Date.now或者performance.now就可以了，但是每次获取一下比较消耗性能，所以 React 设置了currentRendererTime来记录这个值.


currentRendererTime 在任何时候都可以更新，但是currentSchedulerTime只有在没有未完成的工作或者确定工作不在浏览器事件执行中才更新（即nextFlushedExpirationTime为NoWork或Never）

也就是说，当正在渲染或已有待处理的工作时，requestCurrentTime直接返回currentSchedulerTime，否则如果nextFlushedExpirationTime为NoWork或Never，则更新currentRendererTime和currentSchedulerTime。


isRendering 表示是否处于渲染中，isRendering会在performWorkOnRoot的开始设置为true，在结束设置为false，都是同步的。performWorkOnRoot的先进入渲染阶段然后进入提交阶段，react所有的生命周期钩子都是在此执行的。

findHighestPriorityRoot 看名字就知道是要找到优先级最高的root,并且更新nextFlushedRoot和nextFlushedExpirationTime，当没有任务的时候nextFlushedExpirationTime为NoWork。


如果没有任务需要执行，那么重新计算当前时间并返回。
```
recomputeCurrentRendererTime();
```

## computeExpirationForFiber

```
function computeExpirationForFiber(currentTime, fiber) {
  // 获取当前优先级 currentPriorityLevel 缓存
  var priorityLevel = scheduler.unstable_getCurrentPriorityLevel(); 

  var expirationTime = void 0;
  if ((fiber.mode & ConcurrentMode) === NoContext) {
    // 异步模式之外的，优先级设置为同步模式
    expirationTime = Sync;
  } else if (isWorking && !isCommitting$1) {
    // 在render阶段，优先级设置为下次渲染的到期时间
    expirationTime = nextRenderExpirationTime;
  } else {
    // 在commit阶段，根据priorityLevel进行expirationTime更新
    switch (priorityLevel) {
      case scheduler.unstable_ImmediatePriority:
        // 立即执行的任务
        expirationTime = Sync;
        break;
      case scheduler.unstable_UserBlockingPriority:
        // 因用户交互阻塞的优先级
        expirationTime = computeInteractiveExpiration(currentTime);
        break;
      case scheduler.unstable_NormalPriority:
        // 一般优先级，异步更新
        expirationTime = computeAsyncExpiration(currentTime);
        break;
      case scheduler.unstable_LowPriority:
      case scheduler.unstable_IdlePriority:
        // 低优先级或空闲状态
        expirationTime = Never;
        break;
      default:
        invariant(false, 'Unknown priority level. This error is likely caused by a bug in React. Please file an issue.');
    }

    // 避免在渲染树的时候同时去更新已经渲染的树
    if (nextRoot !== null && expirationTime === nextRenderExpirationTime) {
      expirationTime -= 1;
    }
  }

  // 记录下挂起的用户交互任务中expirationTime最短的一个，在需要时同步刷新所有交互式更新
  if (priorityLevel === scheduler.unstable_UserBlockingPriority 
      && (lowestPriorityPendingInteractiveExpirationTime === NoWork 
      || expirationTime < lowestPriorityPendingInteractiveExpirationTime)) {
    lowestPriorityPendingInteractiveExpirationTime = expirationTime;
  }

  return expirationTime;
}
```



## React中各种expirationTime

在 React 的调度过程中存在着非常多不同的expirationTime变量帮助 React 去实现在单线程环境中调度不同优先级的任务这个需求。

值得关注的几个expirationTime有这么几个

####  childExpirationTime

每次一个节点调用setState或者forceUpdate都会产生一个更新并且计算一个expirationTime，那么这个节点的expirationTime就是当时计算出来的值，因为这个更新本身就是由这个节点产生的


因为 React 的更新需要从FiberRoot开始，所以会执行一次向上遍历找到FiberRoot，遍历过程中 React 就会对每一个该节点的父节点链上的节点设置childExpirationTime，因为这个更新是他们的子孙节点造成的，父节点需要进行记录。


在向下更新整棵Fiber树的时候，每个节点都会执行对应的update方法，在这个方法里面就会使用节点本身的expirationTime和childExpirationTime来判断他是否可以直接跳过，不更新子树。expirationTime代表他本身是否有更新，如果他本身有更新，那么他的更新可能会影响子树；childExpirationTime表示他的子树是否产生了更新；如果两个都没有，那么子树是不需要更新的，直接熔断掉```bailoutOnAlreadyFinishedWork```。

```
function bailoutOnAlreadyFinishedWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderExpirationTime: ExpirationTime,
): Fiber | null {
  cancelWorkTimer(workInProgress);

  if (current !== null) {
    // Reuse previous context list
    workInProgress.firstContextDependency = current.firstContextDependency;
  }

  // Check if the children have any pending work.
  const childExpirationTime = workInProgress.childExpirationTime;
  
  // 如果 childExpirationTime 小于当前更新优先级
  // 或者子树无更新
  // 跳过子树的更新 （性能优化项，避免了每次都重新render）
  if (
    childExpirationTime === NoWork ||
    childExpirationTime > renderExpirationTime
  ) {
    return null;
  } else {
    // This fiber doesn't have work, but its subtree does. Clone the child
    // fibers and continue.
    cloneChildFibers(current, workInProgress);
    return workInProgress.child;
  }
}
```

这个childExpirationTime还有一个重要的作用就是，在context中，provider的value发生了变化，会设置consumer的childExpirationTime，以及该consumer父节点链上的节点的childExpirationTime，从而保证组件能正确的更新。


## suspendedTime

## lastestPingedTime

## nextExpirationTimeToWorkOn

## nextFlushedExpirationTime

