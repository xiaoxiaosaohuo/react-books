调度任务从根节点（root）开始，按照深度优先遍历执行，根节点可以理解为ReactDOM.render(<App/>, document.getElementById('root'))所创建，正常来说一个项目应该只有一个
一个调度任务可能会包含多个root，执行完高优先级root，才能执行下一个root，高优先级root也是可以插队的
fiber.expirationTime属性代表该fiber的优先级，数值越大，优先级越高，通过当前时间减超时时间获得，同步任务优先级默认为最高
react当前时间是倒着走的，当前时间初始为一个极大值，随着时间流逝，当前时间越来越小；任务（fiber）的优先级是根据当前时间减超时时间计算，如当前时间10000，任务超时时间500，当前任务优先级算法10000-500=9500；新任务来临，时间流逝，当前时间变为9500，超时时间不变，新任务优先级算法9500-500=9000；新任务优先级低于原任务；

注意：当时间来到9500时，老任务超时，自动获得最高优先级，因为所有新任务除同步任务外优先级永远不会超过老任务，react以这套时间规则来防止低优先级任务一直被插队
fiber.childExpirationTime属性代表该fiber的子节点优先级，该属性可以用来判断fiber的子节点还有没有任务或比较优先级，更新时若没有任务（fiber.childExpirationTime===0）或者本次渲染的优先级大于子节点优先级，那么不必再往下遍历
当某组件（fiber）触发了任务时，会往上遍历，将fiber.return.expirationTime和fiber.return.childExpirationTime全部更新为该组件的expirationTime，一直到root，可以理解为在root上收集更新
fiber有个alternate属性，实际上是fiber的复制体，同时也指向本体，用于react error boundary踩错误，随时回滚，执行完毕且无误后本体与复制体同步；在初始化时，react并不创建alternate，而在更新时创建



animationTick 结合 idleTick 形成消息传递事件的发送方和接收方，同时也分别是 requestAnimationFrame 回调函数和触发函数。通过 messageKey 来识别是否是通知的自己。idleTick 里面的循环判断和 timeRemaining 相同，判断是否有空闲时间，有才进行 callUnsafely，执行 callbac



首先判断currentDidTimeout，currentDidTimeout为false说明任务没有过期，大家要知道过期任务拥有最高优先级，那么即使有更高级的任务依然无法打断，直接return false；
再判断firstCallbackNode.expirationTime < currentExpirationTime，这里实际上是照顾一种特殊的情况，那就是一个最高优先级的任务插入之后，低优先级的任务还在运行中，这种情况是仍然需要打断的；这里firstCallbackNode其实是那个插入的高优先级任务，而currentExpirationTime其实是上一个任务的expirationTime，只是还没结算；
最后是一个shouldYieldToHost()，很简单，就是看任务在帧内是否过期，注意到这边任务帧内过期的话是return true，代表直接就能被打断；



Scheduler.js使用animationTick作为requestAnimationFrame的callback，用以计算frameDeadline和调用传入的回调函数，在react中即为调度函数；frameDeadline表示的是运行到当前帧的帧过期时间，计算方法是当前时间 + activeFrameTime，activeFrameTime表示的是一帧的时间，默认为33ms，但是会根据设备动态调整，比如在刷新频率更高的设备上，连续运行两帧的当前时间比运行到该帧的过期时间frameDeadline都小，说明我们一帧中的js任务耗时也小，一帧时间充足且requestAnimationFrame调用比预设的33ms频繁，那么activeFrameTime会降低以达到最佳性能

有了frameDeadline与用户自定义的过期时间timeoutTime，那么我们很容易得到polyfill requestIdleCallback的原理：用户定义的callback在这一帧有空就去运行，超过帧过期时间frameDeadline就到下一帧去运行，你可以超过帧过期时间，但是你不能超过用户定义的timeoutTime，一旦超过，我啥也不管，直接运行callback



Scheduler.js将每一次unstable_scheduleCallback的调用根据用户定义的timeout来为任务分配优先级，timeout越小，优先级越高。具体实现为：用双向链表结构来表示任务列表，且按优先级从高到低的顺序进行排列，当某个任务插入时，从头结点开始循环遍历，若遇到某个任务结点node的expirationTime > 插入任务的expirationTime，说明插入任务比node优先级高，则退出循环，并在node前插入，expirationTime = 当前时间 + timeout；这样就实现了按优先级排序的任务插入功能，animationTick会循环调用这些任务链表。



https://misterye.com/archives/6850.html



```
过期时间，与任务单元的优先级相关，根据expirationTime来判断是否进行下一个分片任务，过期时间决定了更新的批处理方式，所以我们希望在同一浏览器事件中发生的具有相同优先级的所有更新都接受相同的过期时间。代码中先通过computeExpirationForFiber函数根据不同的阶段计算expirationTime，再使用scheduleWork函数更新任务队列，可能的初始值如下：

var NoWork = 0; // 没有任务等待处理
var Never = 1; // 暂不执行，优先级最低
var Sync = maxSigned31BitInt; // 同步模式，立即处理，优先级最高


computeExpirationForFiber


currentTime是requestCurrentTime函数的返回值，currentTime计算是基于起始时间（scheduler.unstable_now() -originalStartTimeMs）计算的。react追踪两个单独的时间，"renderer" time（currentRendererTime）和"scheduler" time（currentSchedulerTime）。"renderer" time可以随时更新，目前他只是为了降低调用性能；"scheduler" time只有在没有未完成的工作或者确定工作不在浏览器事件执行中才更新（即nextFlushedExpirationTime为NoWork或Never）。也就是说，当正在渲染或已有待处理的工作时，requestCurrentTime直接返回currentSchedulerTime，否则如果nextFlushedExpirationTime为NoWork或Never，则更新currentRendererTime和currentSchedulerTime。
fiber：需要处理的fiber node



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

值得注意的是computeInteractiveExpiration和computeAsyncExpiration都调用了computeExpirationBucket，只是传入不一样参数以区分优先级高低（Interactive优先级高于Async），调用如下：

// 返回距离num最近的precision的倍数
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

var HIGH_PRIORITY_EXPIRATION = 500;
var HIGH_PRIORITY_BATCH_SIZE = 100;

function computeInteractiveExpiration(currentTime) {
  return computeExpirationBucket(currentTime, HIGH_PRIORITY_EXPIRATION, HIGH_PRIORITY_BATCH_SIZE);
}
```
