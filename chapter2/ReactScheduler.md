## React-Scheduler

Fiber给不同任务分配了不同的优先级

```
var ImmediatePriority = 1;
var UserBlockingPriority = 2;
var NormalPriority = 3;
var LowPriority = 4;
var IdlePriority = 5;
```
不同优先级的任务如何按一定次序执行，如何防止低优先级的任务不被饿死，这就是React-Scheduler要干的事情。

## window.requestAnimationFrame

window.requestAnimationFrame() 告诉浏览器——你希望执行一个动画，并且要求浏览器在下次重绘之前调用指定的回调函数更新动画。该方法需要传入一个回调函数作为参数，该回调函数会在浏览器下一次重绘之前执行

```
window.requestAnimationFrame(callback);

下一次重绘之前更新动画帧所调用的函数(即上面所说的回调函数)。callback回调函数会被传入DOMHighResTimeStamp参数，该参数与performance.now()的返回值相同，它表示requestAnimationFrame() 开始去执行回调函数的时刻。
```

为了提高性能和电池寿命，因此在大多数浏览器里，当requestAnimationFrame() 运行在后台标签页或者隐藏的```<iframe> ```里时，requestAnimationFrame() 会被暂停调用以提升性能和电池寿命


## window.requestIdleCallback

window.requestIdleCallback()会在浏览器空闲时期依次调用函数， 这就可以让开发者在主事件循环中执行后台或低优先级的任务，而且不会对像动画和用户交互这样延迟敏感的事件产生影响。函数一般会按先进先调用的顺序执行，然而，如果回调函数指定了执行超时时间timeout，则有可能为了在超时前执行函数而打乱执行顺序

##### 参数

- callback
- options 
  
callback函数会接收到一个名为 IdleDeadline 的参数，这个参数可以获取当前空闲时间以及回调是否在超时时间前已经执行的状态，也就是可以让你判断用户代理(浏览器)还剩余多少闲置时间可以用来执行耗时任务
- IdleDeadline.didTimeout 一个Boolean类型当它的值为true的时候说明callback正在被执行(并且上一次执行回调函数执行的时候由于时间超时回调函数得不到执行)，因为在执行requestIdleCallback回调的时候指定了超时时间并且时间已经超时。
- IdleDeadline.timeRemaining() 返回一个时间DOMHighResTimeStamp, 并且是浮点类型的数值，它用来表示当前闲置周期的预估剩余毫秒数。如果idle period已经结束，则它的值是0。你的回调函数(传给requestIdleCallback的函数)可以重复的访问这个属性用来判断当前线程的闲置时间是否可以在结束前执行更多的任务。

在输入处理之后，已完成给定帧的渲染和合成，主线程就空闲了，直到下一帧开始，下一个待处理的任务开始运行。那么空闲时段就是在给定帧渲染到屏幕和下一帧开始处理之间的时间

[后台任务的协同调度](https://developer.mozilla.org/en-US/docs/Web/API/Background_Tasks_API)


![结构](../images/image01.png)

## MessageChannel

Channel Messaging API的Channel Messaging接口允许我们创建一个新的消息通道，并通过它的两个MessagePort 属性发送数据

```
var channel = new MessageChannel();
var para = document.querySelector('p');
    
var ifr = document.querySelector('iframe');
var otherWindow = ifr.contentWindow;

ifr.addEventListener("load", iframeLoaded, false);
    
function iframeLoaded() {
  otherWindow.postMessage('Hello from the main page!', '*', [channel.port2]);
}

channel.port1.onmessage = handleMessage;
function handleMessage(e) {
  para.innerHTML = e.data;
}
```

onmessage的回调函数的调用时机是在一帧的paint完成之后




## 链表

React中有大量的链表来存储数据，React-Scheduler使用了双向循环链表结构


- 双向链表的每个节点都有previous和next指针，分别指向前一个和后一个节点
- 最后一个节点的next指针指向首节点，首节点的previous指针指向最后一个节点

假设任务有优先级高低之分，高的在前面，低的在后面，不断有新任务进来，也有任务会完成，就需要插入任务和删除任务。
```
//首节点，一开始链表里没有节点
let firstTask = null ;
let expirationTime =  performance.now() + timeout;
const task = {
    expirationTime:expirationTime,
    priorityLevel,
    next:null,
    previous:null
}

function deleteTask(task){
    const next = task.next;
    //只有一个节点
    if (next === task) {
        firstTask = null;
    }else{
        //要删除的节点是第一个节点
        if (task === firstTask) {
            firstTask = next;
        }
        const previous = task.previous;
        previous.next = next;
        next.previous = previous;
    }
    task.next = task.previous = null;

}

function addTask(newTask,expirationTime){
    //list中的第一个节点
    if (firstTask === null) {
        firstTask = newTask.next = newTask.previous = newTask;
    }else{
        // 已经有其他任务了，要找到合适的位置插入
        let next = null ;//记录要插到哪个位置的前面
        let task = firstTask; //保存第一个节点,并从第一个节点往后遍历

        do{
            if (expirationTime < task.expirationTime) {
                // 新任务的早于task过期
                next = task;
            }
            task = task.next;
        }while(task !== firstTask)

        if (next === null) {
            //找了一圈没找到，说明应该放到链表的最后
            next = firstTask;
        }else if (next === firstTask) {
            // 新任务最早过期
            firstTask = newTask;
        }
        let previous = next.previous;
        previous.next = next.previous = newTask;
        newTask.next = next;
        newTask.previous = previous;
    }



}

```

## 任务优先级


上面介绍了五种优先级，数字越小优先级越高。这五种优先级对应着过期时间

```
// Times out immediately
var IMMEDIATE_PRIORITY_TIMEOUT = -1;
// Eventually times out
var USER_BLOCKING_PRIORITY = 250;
var NORMAL_PRIORITY_TIMEOUT = 5000;
var LOW_PRIORITY_TIMEOUT = 10000;
// Never times out
var IDLE_PRIORITY = maxSigned31BitInt;
```


每个任务在添加到链表里的时候，都会通过 performance.now() + timeout来得出这个任务的过期时间，随着时间的推移，当前时间会越来越接近这个过期时间，所以过期时间越小的代表优先级越高。如果过期时间已经比当前时间小了，说明这个任务已经过期了还没执行，需要立马去执行。

##### unstable_scheduleCallback

- unstable_scheduleCallback方法用来对任务进行排序的。
- timeoutForPriorityLevel 方法根据priorityLevel返回上面提到的timeout时间
- 生成一个task ，插入新任务
- requestHostCallback 和 requestHostTimeout 执行任务
  

```
function unstable_scheduleCallback(priorityLevel, callback, options) {
  var currentTime = getCurrentTime();

  var startTime;
  var timeout;
  //根据option的参数计算过期时间
  if (typeof options === 'object' && options !== null) {
    var delay = options.delay;
    if (typeof delay === 'number' && delay > 0) {
      startTime = currentTime + delay;
    } else {
      startTime = currentTime;
    }
    timeout =
      typeof options.timeout === 'number'
        ? options.timeout
        : timeoutForPriorityLevel(priorityLevel);
  } else {
    timeout = timeoutForPriorityLevel(priorityLevel);
    startTime = currentTime;
  }

  var expirationTime = startTime + timeout;
    //生成一个task
  var newTask = {
    callback,
    priorityLevel,
    startTime,
    expirationTime,
    next: null,
    previous: null,
  };
    //这是个延迟的任务
  if (startTime > currentTime) {
    // This is a delayed task.
    insertDelayedTask(newTask, startTime);
    if (firstTask === null && firstDelayedTask === newTask) {
      // All tasks are delayed, and this is the task with the earliest delay.
      if (isHostTimeoutScheduled) {
        // Cancel an existing timeout.
        cancelHostTimeout();
      } else {
        isHostTimeoutScheduled = true;
      }
      // Schedule a timeout.
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
      //插入任务
    insertScheduledTask(newTask, expirationTime);
    // Schedule a host callback, if needed. If we're already performing work,
    // wait until the next time we yield.
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true;
      requestHostCallback(flushWork);
    }
  }

  return newTask;
}

```

接下来介绍requestHostCallback 和requestHostTimeout 在这之前再说下requestIdleCallback;


React利用requestAnimationFrame和MessageChannel pollyfill了一个requestIdleCallback

## requestIdleCallback pollyfill

DOM Scheduler实现类似于requestIdleCallback,他是通过调度requestAnimationFrame来工作的，
存储一帧的开始时间，然后调用postMessage，它在绘制后进行调度，在postMessage的处理方法中尽可能多的工作，
直到当前帧的时间用完，这样可确保布局，绘制和其他浏览器的工作都能计入可用时间范围。

帧率是可以动态调整的。

##### requestAnimationFrameWithTimeout

由于requestAnimationFrame当天页面tab切换到后台的时候不会继续运行，所以启用了100ms 的 setTimeout。

```
const ANIMATION_FRAME_TIMEOUT = 100;
let rAFID;
let rAFTimeoutID;
const requestAnimationFrameWithTimeout = function(callback) {
  // schedule rAF and also a setTimeout
  rAFID = localRequestAnimationFrame(function(timestamp) {
    // cancel the setTimeout
    localClearTimeout(rAFTimeoutID);
    callback(timestamp);
  });
  rAFTimeoutID = localSetTimeout(function() {
    // cancel the requestAnimationFrame
    localCancelAnimationFrame(rAFID);
    callback(getCurrentTime());
  }, ANIMATION_FRAME_TIMEOUT);
};
```

- 当我们调用requestAnimationFrameWithTimeout并传入一个callback的时候，会同时启动一个requestAnimationFrame和一个setTimeout,两者都会去执行callback。
- 由于requestAnimationFrame执行优先级相对较高，它内部会调用localClearTimeout取消下面定时器的操作。所以在页面active情况下的表现跟requestAnimationFrame是一致的。



## 任务执行的流程

1. - 在每一帧开始的rAF的回调里记录每一帧的开始时间，并计算每一帧的过期时间
2. - 通过messageChannel发送消息
3. - 在帧末messageChannel的回调里接收消息，根据当前帧的过期时间和当前时间进行比对来决定当前帧能否执行任务，可以的话依次从任务链表里拿出队首任务来执行，执行尽可能多的任务后如果还有任务，下一帧再重新调度
  
##### 执行任务前期准备

声明变量

```
  let scheduledHostCallback = null; //任务链表的执行器,也就是requestHostCallback的参数flushWork
  let isMessageEventScheduled = false;//// 消息事件是否执行

  let isAnimationFrameScheduled = false;

  let timeoutID = -1;

  let frameDeadline = 0; //代表一帧的过期时间，通过rAF回调入参t加上activeFrameTime来计算
  
  //开始假设以30fps的帧率运行，如果能达到更流程的帧率，启发式跟踪算法会调整到更快。
 
  let previousFrameTime = 33; // 一帧的时间: 1000 / 30 ≈ 33
  let activeFrameTime = 33;
  let fpsLocked = false;
  let maxFrameLength = 300; //最大帧时长
  let needsPaint = false;
```

### 动态调整帧率策略

- 用当前每帧的总时间与实际每帧的时间进行比较， 当实际时间小于当前时间且稳定( 前后两次都小于当前时间 )， 那么就会认为这个值是有效的， 然后将每帧时间调整为该值（取前后两次中时间大的值）。这是Fiber在进行动态的压帧. 根据性能提供最优的时间。

- requestAnimationFrame 让一批扁平任务恰好控制在一块一块的33ms这样的时间片内执行

### 创建一个消息信道
通过MessageChannel创建消息信道
```
// We use the postMessage trick to defer idle work until after the repaint.
  const channel = new MessageChannel();
  //port2用来发消息
  const port = channel.port2;
  //port2用来监听消息做任务调度的具体工作

```
### 1. 计算每一帧的截止时间 

下面animationTick会传入requestAnimationFrameWithTimeout作为回调方法，会在这个方法里计算时间


```

const animationTick = function(rafTime) {
    //如果存在调度器回调函数则在一帧的开头急切地安排下一帧的动画回调
    //如果在帧的后半段安排动画回调的话, 就会增大下一帧超过 100ms 的几率
    //从而会浪费一个帧的利用,
    if (scheduledHostCallback !== null) {
      
      requestAnimationFrameWithTimeout(animationTick);
    } else {
      //无任务直接退出
      isAnimationFrameScheduled = false;
      return;
    }
    // rafTime 就是 requestAnimationFrame 回调接收的时间
    //用连续的两次时间 被不断的压缩activeFrameTime
    let nextFrameTime = rafTime - frameDeadline + activeFrameTime;
    if (
      nextFrameTime < activeFrameTime &&
      previousFrameTime < activeFrameTime &&
      !fpsLocked
    ) {
      if (nextFrameTime < 8) {
          //不支持高于120hz的帧率
        nextFrameTime = 8;
      }
      //如果两个帧连续短，那么这表明我们
      //实际上帧速率高于我们目前优化的帧速率。
      activeFrameTime =
        nextFrameTime < previousFrameTime ? previousFrameTime : nextFrameTime;
    } else {
      previousFrameTime = nextFrameTime;
    }
    //计算当前帧的截止时间，用开始时间加上每一帧的渲染时间
    frameDeadline = rafTime + activeFrameTime;
    if (!isMessageEventScheduled) {
      isMessageEventScheduled = true;
      port.postMessage(undefined);
    }
  };
```
### 2. postMessage 发送消息
发送消息


### 3. onmessage 触发监听回调

调用scheduledHostCallback
```
channel.port1.onmessage = function(event) {
    isMessageEventScheduled = false;
    if (scheduledHostCallback !== null) {
      const currentTime = getCurrentTime();
      //还有剩余时间
      const hasTimeRemaining = frameDeadline - currentTime > 0;
      try {
        const hasMoreWork = scheduledHostCallback(
          hasTimeRemaining,
          currentTime,
        );
       //还有其他的工作，意味着上一帧没有处理完
        if (hasMoreWork) {
          // Ensure the next frame is scheduled.
          if (!isAnimationFrameScheduled) {
            isAnimationFrameScheduled = true;
            requestAnimationFrameWithTimeout(animationTick);
          }
        } else {
          scheduledHostCallback = null;
        }
      } catch (error) {
        // If a scheduler task throws, exit the current browser task so the
        // error can be observed, and post a new task as soon as possible
        // so we can continue where we left off.
        isMessageEventScheduled = true;
        port.postMessage(undefined);
        throw error;
      }
      // Yielding to the browser will give it a chance to paint, so we can
      // reset this.
      needsPaint = false;
    }
}
```

### 4. 执行任务

scheduledHostCallback就是下面的flushWork
- hasTimeRemaining 是否还有剩余时间
- initialTime 当前时间


```
function flushWork(hasTimeRemaining, initialTime) {


  let currentTime = initialTime;
  //调度被延迟的任务，如果起startTime小于currentTime,就将其插入到任务列表
  advanceTimers(currentTime);

  isPerformingWork = true;
  try {
      //没有剩余时间，表示过期了。
    if (!hasTimeRemaining) {
      
      while (
        firstTask !== null &&
        firstTask.expirationTime <= currentTime && //队首任务时间比当前时间小，说明过期了
        !(enableSchedulerDebugging && isSchedulerPaused)
      ) {
        // 执行队首任务，把队首任务从链表移除，并把第二个任务置为队首任务。执行任务可能产生新的任务，再把新任务插入到任务链表
        flushTask(firstTask, currentTime);
        currentTime = getCurrentTime();
        advanceTimers(currentTime);
      }
    } else {
        //当前帧有富余时间，运行任务，知道当前帧到期
      if (firstTask !== null) {
        do {
          flushTask(firstTask, currentTime);
          currentTime = getCurrentTime();
          advanceTimers(currentTime);
        } while (
          firstTask !== null &&
          !shouldYieldToHost() && //shouldYieldToHost 当前帧过期标志
          !(enableSchedulerDebugging && isSchedulerPaused)
        );
      }
    }
    // Return whether there's additional work
    if (firstTask !== null) {
      return true;
    } else {
      if (firstDelayedTask !== null) {
        requestHostTimeout(
          handleTimeout,
          firstDelayedTask.startTime - currentTime,
        );
      }
      return false;
    }
  } finally {
    isPerformingWork = false;
  }
}

```

##### shouldYieldToHost

```
shouldYieldToHost = function() {
    // 当前帧的截止时间比当前时间小则为true，代表当前帧过期了
     return getCurrentTime() >= frameDeadline;
};

```

### 总结一下流程

- 任务根据优先级和任务产生时的当前时间来确定过期时间
- 任务根据过期时间加入任务链表
- 启动任务的调度，1是任务链表从无到有时，2是任务链表加入了新的最高优先级任务时
- requestAnimationFrameWithTimeout和messageChannel来实现任务调度
- requestAnimationFrameWithTimeout的回调函数计算每一帧的时间，截止时间，并通过postMessage触发任务调用
- messageChannel在监听消息的回调中调用scheduledHostCallback
- scheduledHostCallback根据是否有剩余时间、当前时间、任务链表第一个任务的过期时间来决定当前帧是否执行任务
- 任务过期的话就会把任务链表内过期的任务都执行一遍直到没有过期任务或者没有任务
- 任务没过期的话，则会在当前帧过期之前尽可能多的执行任务，如果还有任务（hasMoreWork）就继续调度