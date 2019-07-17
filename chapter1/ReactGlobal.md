## ReactFiberScheduler.js 中的变量


```
// Used to ensure computeUniqueAsyncExpiration is monotonically decreasing.
let lastUniqueAsyncExpiration: number = Sync - 1;

let isWorking: boolean = false;

// The next work in progress fiber that we're currently working on.
let nextUnitOfWork: Fiber | null = null;
let nextRoot: FiberRoot | null = null;
// The time at which we're currently rendering work.
let nextRenderExpirationTime: ExpirationTime = NoWork;
let nextLatestAbsoluteTimeoutMs: number = -1;
let nextRenderDidError: boolean = false;

// The next fiber with an effect that we're currently committing.
let nextEffect: Fiber | null = null;

let isCommitting: boolean = false;
let rootWithPendingPassiveEffects: FiberRoot | null = null;
let passiveEffectCallbackHandle: * = null;
let passiveEffectCallback: * = null;

let legacyErrorBoundariesThatAlreadyFailed: Set<mixed> | null = null;

// Used for performance tracking.
let interruptedBy: Fiber | null = null;

let stashedWorkInProgressProperties;
let replayUnitOfWork;
let mayReplayFailedUnitOfWork;
let isReplayingFailedUnitOfWork;
let originalReplayError;
let rethrowOriginalError;
```

介绍一下几个重点需要关注的变量

#### isWorking

commitRoot和renderRoot开始都会设置为true，然后在他们各自阶段结束的时候都重置为false

用来标志是否当前有更新正在进行，不区分阶段

#### isCommitting

commitRoot开头设置为true，结束之后设置为false

用来标志是否处于commit阶段


#### nextUnitOfWork

用于记录render阶段Fiber树遍历过程中下一个需要执行的节点。

在resetStack中分别被重置

他只会指向workInProgress


#### nextRoot & nextRenderExpirationTime

用于记录下一个将要渲染的root节点和下一个要渲染的任务的

#### nextEffect

用于commit阶段记录firstEffect -> lastEffect链遍历过程中的每一个Fiber




```
// Linked-list of roots
let firstScheduledRoot: FiberRoot | null = null;
let lastScheduledRoot: FiberRoot | null = null;

let callbackExpirationTime: ExpirationTime = NoWork;
let callbackID: *;
let isRendering: boolean = false;
let nextFlushedRoot: FiberRoot | null = null;
let nextFlushedExpirationTime: ExpirationTime = NoWork;
let lowestPriorityPendingInteractiveExpirationTime: ExpirationTime = NoWork;
let hasUnhandledError: boolean = false;
let unhandledError: mixed | null = null;

let isBatchingUpdates: boolean = false;
let isUnbatchingUpdates: boolean = false;

let completedBatches: Array<Batch> | null = null;

let originalStartTimeMs: number = now();
let currentRendererTime: ExpirationTime = msToExpirationTime(
  originalStartTimeMs,
);
let currentSchedulerTime: ExpirationTime = currentRendererTime;

// Use these to prevent an infinite loop of nested updates
const NESTED_UPDATE_LIMIT = 50;
let nestedUpdateCount: number = 0;
let lastCommittedRootDuringThisBatch: FiberRoot | null = null;
```


#### firstScheduledRoot & lastScheduledRoot
用于存放有任务的所有root的单链表结构

- 在findHighestPriorityRoot用来检索优先级最高的root
- 在addRootToSchedule中会修改


#### callbackExpirationTime & callbackID

记录请求ReactScheduler的时候用的过期时间，如果在一次调度期间有新的调度请求进来了，而且优先级更高，那么需要取消上一次请求，如果更低则无需再次请求调度。

callbackID是ReactScheduler返回的用于取消调度的 ID


#### isRendering

performWorkOnRoot开始设置为true，结束的时候设置为false，表示进入渲染阶段，这是包含render和commit阶段的

#### nextFlushedRoot & nextFlushedExpirationTime
用来标志下一个需要渲染的root和对应的expirtaionTime

#### deadline & deadlineDidExpire
deadline是ReactScheduler中返回的时间片调度信息对象

用于记录是否时间片调度是否过期，在shouldYield根据deadline是否过期来设置


#### isBatchingUpdates & isUnbatchingUpdates & isBatchingInteractiveUpdates


batchedUpdates、unBatchedUpdates，deferredUpdates、interactiveUpdates等这些方法用来存储更新产生的上下文的变量


#### originalStartTimeMs
固定值，js 加载完一开始计算的时间戳

#### currentRendererTime & currentSchedulerTime

计算从页面加载到现在为止的毫秒数，后者会在isRendering === true的时候用作固定值返回，不然每次requestCurrentTime都会重新计算新的时间。

