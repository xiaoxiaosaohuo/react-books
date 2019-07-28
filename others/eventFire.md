## 事件分发和回收
经过以前的分析当事件触发的时候会调用 `dispatchEvent`

#### dispatchEvent

- topLevelType React中的事件名，如onClick
- nativeEvent DOM 事件对象


主要过程：
1. 找到触发事件的原始节点的fiber对象
2. 生成bookKeeping对象，包含了事件名称，原始事件对象，以及最近的Fiber对象
3. 调用batchedUpdates
4. 调用handleTopLevel,其核心是生成合成事件对象和合并事件
5. runEventsInBatch 事件批处理


```
export function dispatchEvent(
  topLevelType: DOMTopLevelEventType,
  eventSystemFlags: EventSystemFlags,
  nativeEvent: AnyNativeEvent,
): void {
  if (!_enabled) {
    return;
  }

  const nativeEventTarget = getEventTarget(nativeEvent);
  // 找到事件触发的原始节点最近的Fiber对象
  //根据设置在DOM节点上的`internalInstanceKey`来寻找
  let targetInst = getClosestInstanceFromNode(nativeEventTarget);

  if (
    targetInst !== null &&
    typeof targetInst.tag === 'number' &&
    !isFiberMounted(targetInst)
  ) {
    targetInst = null;
  }

  if (enableFlareAPI) {
    if (eventSystemFlags === PLUGIN_EVENT_SYSTEM) {
      dispatchEventForPluginEventSystem(
        topLevelType,
        eventSystemFlags,
        nativeEvent,
        targetInst,
      );
    } else {
      // React Flare event system
      dispatchEventForResponderEventSystem(
        (topLevelType: any),
        targetInst,
        nativeEvent,
        nativeEventTarget,
        eventSystemFlags,
      );
    }
  } else {
    dispatchEventForPluginEventSystem(
      topLevelType,
      eventSystemFlags,
      nativeEvent,
      targetInst,
    );
  }
}

function dispatchEventForPluginEventSystem(
  topLevelType: DOMTopLevelEventType,
  eventSystemFlags: EventSystemFlags,
  nativeEvent: AnyNativeEvent,
  targetInst: null | Fiber,
): void {
    //生成bookKeeping对象，包含了事件名称，原始事件对象，以及最近的Fiber对象
  const bookKeeping = getTopLevelCallbackBookKeeping(
    topLevelType,
    nativeEvent,
    targetInst,
  );

  try {
    // Event queue being processed in the same cycle allows
    // `preventDefault`.
    batchedEventUpdates(handleTopLevel, bookKeeping);
  } finally {
    releaseTopLevelCallbackBookKeeping(bookKeeping);
  }
}


export function batchedUpdates(fn, bookkeeping) {
  if (isInsideEventHandler) {
    // If we are currently inside another batch, we need to wait until it
    // fully completes before restoring state.
    return fn(bookkeeping);
  }
  isInsideEventHandler = true;
  try {
    return batchedUpdatesImpl(fn, bookkeeping);
  } finally {
    isInsideEventHandler = false;
    finishEventHandler();
  }
}
let batchedUpdatesImpl = function(fn, bookkeeping) {
  return fn(bookkeeping);
};
```

handleTopLevel中根据当前的targetInst向上遍历获取所有父组件ancestors数组，接下来遍历ancestors，执行runExtractedEventsInBatch方法

```
function handleTopLevel(bookKeeping: BookKeepingInstance) {
  let targetInst = bookKeeping.targetInst;

  let ancestor = targetInst;
  do {
    if (!ancestor) {
      const ancestors = bookKeeping.ancestors;
      ((ancestors: any): Array<Fiber | null>).push(ancestor);
      break;
    }
    const root = findRootContainerNode(ancestor);
    if (!root) {
      break;
    }
    bookKeeping.ancestors.push(ancestor);
    //找打最近的react node实例
    ancestor = getClosestInstanceFromNode(root);
  } while (ancestor);

  for (let i = 0; i < bookKeeping.ancestors.length; i++) {
    targetInst = bookKeeping.ancestors[i];
    const eventTarget = getEventTarget(bookKeeping.nativeEvent);
    const topLevelType = ((bookKeeping.topLevelType: any): DOMTopLevelEventType);
    const nativeEvent = ((bookKeeping.nativeEvent: any): AnyNativeEvent);

    runExtractedPluginEventsInBatch(
      topLevelType,
      targetInst,
      nativeEvent,
      eventTarget,
    );
  }
}
```
runExtractedPluginEventsInBatch和extractPluginEvents的主要目的是生成事件对象，调用accumulateTwoPhaseDispatches合并事件。


runEventsInBatch用于批处理 extractEvents构造出的合成事件


```
function extractPluginEvents(
  topLevelType: TopLevelType,
  targetInst: null | Fiber,
  nativeEvent: AnyNativeEvent,
  nativeEventTarget: EventTarget,
): Array<ReactSyntheticEvent> | ReactSyntheticEvent | null {
  let events = null;
  for (let i = 0; i < plugins.length; i++) {
    // Not every plugin in the ordering may be loaded at runtime.
    const possiblePlugin: PluginModule<AnyNativeEvent> = plugins[i];
    if (possiblePlugin) {
      const extractedEvents = possiblePlugin.extractEvents(
        topLevelType,
        targetInst,
        nativeEvent,
        nativeEventTarget,
      );
      if (extractedEvents) {
        events = accumulateInto(events, extractedEvents);
      }
    }
  }
  return events;
}

export function runExtractedPluginEventsInBatch(
  topLevelType: TopLevelType,
  targetInst: null | Fiber,
  nativeEvent: AnyNativeEvent,
  nativeEventTarget: EventTarget,
) {
  const events = extractPluginEvents(
    topLevelType,
    targetInst,
    nativeEvent,
    nativeEventTarget,
  );
  runEventsInBatch(events);
}
```

#### 构造合成事件

```
function extractEvents(topLevelType, targetInst, nativeEvent, nativeEventTarget) {
  var events = null;
  for (var i = 0; i < plugins.length; i++) {
    // Not every plugin in the ordering may be loaded at runtime.
    var possiblePlugin = plugins[i];
    if (possiblePlugin) {
      var extractedEvents = possiblePlugin.extractEvents(topLevelType, targetInst, nativeEvent, nativeEventTarget);
      if (extractedEvents) {
        events = accumulateInto(events, extractedEvents);
      }
    }
  }
  return events;
```


遍历plugins，调用每个插件的extractEvents方法，找到对应的事件构造器，

```
case  case TOP_CLICK::
        EventConstructor = SyntheticMouseEvent;
        break;
        
        ....
        
   //省略一段代码
   
   
 const event = EventConstructor.getPooled(
  dispatchConfig,
  targetInst,
  nativeEvent,
  nativeEventTarget,
);
accumulateTwoPhaseDispatches(event);
return event;
```

getPooled前面提到过。构造完事件对象之后，调用 accumulateTwoPhaseDispatches方法，其中会调用traverseTwoPhase


#### traverseTwoPhase

向上遍历当前fiber node ，存储在path中。然后分别对捕获和冒泡阶段进行处理。

```
function traverseTwoPhase(inst, fn, arg) {
  var path = [];
  while (inst) {
    path.push(inst);
    inst = getParent(inst);
  }
  var i = void 0;
  //捕获阶段
  for (i = path.length; i-- > 0;) {
    fn(path[i], 'captured', arg);
  }
  //冒泡阶段
  for (i = 0; i < path.length; i++) {
    fn(path[i], 'bubbled', arg);
  }
}
```

参数fn就是下面的accumulateDirectionalDispatches

```
function accumulateDirectionalDispatches(inst, phase, event) {
  {
    !inst ? warningWithoutStack$1(false, 'Dispatching inst must not be null') : void 0;
  }
  var listener = listenerAtPhase(inst, event, phase);
  if (listener) {
    event._dispatchListeners = accumulateInto(event._dispatchListeners, listener);
    event._dispatchInstances = accumulateInto(event._dispatchInstances, inst);
  }
}
```
listenerAtPhase 就是专门从当前的fibernode 上获取事件回调的。
取到回调之后，保存到事件event的 _dispatchListeners属性上，并且将当前元素及其父元素的FiberNode 保存到event的 _dispatchInstances属性上.

拿到所有与事件相关的元素实例以及事件的回调函数之后，就可以对合成事件进行批量处理了。


#### 事件处理

runEventsInBatch 事件批处理的入口
```
function runEventsInBatch(events, simulated) {
  if (events !== null) {
    eventQueue = accumulateInto(eventQueue, events);
  }

  // Set `eventQueue` to null before processing it so that we can tell if more
  // events get enqueued while processing.
  var processingEventQueue = eventQueue;
  eventQueue = null;

  if (!processingEventQueue) {
    return;
  }

  if (simulated) {
    forEachAccumulated(processingEventQueue, executeDispatchesAndReleaseSimulated);
  } else {
    forEachAccumulated(processingEventQueue, executeDispatchesAndReleaseTopLevel);
  }
  !!eventQueue ? invariant(false, 'processEventQueue(): Additional events were enqueued while processing an event queue. Support for this has not yet been implemented.') : void 0;
  // This would be a good time to rethrow if any of the event handlers threw.
  rethrowCaughtError();
}

function forEachAccumulated(arr, cb, scope) {
  if (Array.isArray(arr)) {
    arr.forEach(cb, scope);
  } else if (arr) {
    cb.call(scope, arr);
  }
}
```

首先会将当前需要处理的 events事件，与之前没有处理完毕的队列调用 accumulateInto方法按照顺序进行合并，组合成一个新的队列，因为之前可能就存在还没处理完的合成事件，这里就可以执行了。


```
var executeDispatchesAndReleaseTopLevel = function (e) {
  return executeDispatchesAndRelease(e, false);
};


var executeDispatchesAndRelease = function (event, simulated) {
  if (event) {
    executeDispatchesInOrder(event, simulated);

    if (!event.isPersistent()) {
      event.constructor.release(event);
    }
  }
};

function executeDispatchesInOrder(event, simulated) {
  var dispatchListeners = event._dispatchListeners;
  var dispatchInstances = event._dispatchInstances;
  {
    validateEventDispatches(event);
  }
  if (Array.isArray(dispatchListeners)) {
    for (var i = 0; i < dispatchListeners.length; i++) {
      if (event.isPropagationStopped()) {
        break;
      }
      // Listeners and Instances are two parallel arrays that are always in sync.
      executeDispatch(event, simulated, dispatchListeners[i], dispatchInstances[i]);
    }
  } else if (dispatchListeners) {
    executeDispatch(event, simulated, dispatchListeners, dispatchInstances);
  }
  event._dispatchListeners = null;
  event._dispatchInstances = null;
}


function executeDispatch(event, listener, inst) {
  var type = event.type || 'unknown-event';
  event.currentTarget = getNodeFromInstance(inst);
  invokeGuardedCallbackAndCatchFirstError(type, listener, undefined, event);
  event.currentTarget = null;
}

```

遍历 dispatchListeners ，如果event.isPropagationStopped()为 true，则遍历的循环直接 break掉，中断合成事件的向上遍历执行，也就起到了和原生事件调用 stopPropagation相同的效果。如果循环没有被中断，则继续执行 executeDispatch方法。最终会调用invokeGuardedCallbackAndCatchFirstError 触发回调函数


#### 事件回收

事件执行完毕之后就会做一些清理的工作了。

```
event.currentTarget = null;
event._dispatchListeners = null;
event._dispatchInstances = null;
```

然后回到executeDispatchesAndRelease方法中，

```
if (!event.isPersistent()) {
    event.constructor.release(event);
}
```

如果不持有事件对象的话就释放事件，调用

```
function releasePooledEvent(event) {
  var EventConstructor = this;
  !(event instanceof EventConstructor) ? invariant(false, 'Trying to release an event instance into a pool of a different type.') : void 0;
  event.destructor();
  if (EventConstructor.eventPool.length < EVENT_POOL_SIZE) {
    EventConstructor.eventPool.push(event);
  }
}
```

首先释放掉 event上属性占用的内存，然后把清理后的 event对象再放入对象池中，可以被后续事件对象二次利用。