```
enqueueSetState(inst, payload, callback) {
  //获取实例对应的fiber
  const fiber = ReactInstanceMap.get(inst);
  //计算的当前时间:react确保了在同一个时间中所有的更新都是相同的到期时间
  const currentTime = requestCurrentTime();
  //根据当前时间与fiber计算到期时间
  const expirationTime = computeExpirationForFiber(currentTime, fiber);
  //传入一个到期时间，返回一个对象，见packages\react-reconciler\src\ReactUpdateQueue.js
  const update = createUpdate(expirationTime);
  //利用新的state修改update.payload
  update.payload = payload;
  //利用callback修改update.callback
  if (callback !== undefined && callback !== null) {
    if (__DEV__) {
      warnOnInvalidCallback(callback, 'setState');
    }
    update.callback = callback;
  }
  //？？？存疑，调用scheduler删除当前某个任务，这里先不讨论
  flushPassiveEffects();
  //开始调度setState带来的更新
  enqueueUpdate(fiber, update);
  //开始调度新的当前fiber子树，设置相关的到期时间等等,这里关键是会给当前fiber设置expirationTime为setState的更新任务相同的值
  scheduleWork(fiber, expirationTime);
}

```


```
export function enqueueUpdate<State>(fiber: Fiber, update: Update<State>) {
  // Update queues are created lazily.
  // 使用alternate属性双向连接一个当前fiber和其work-in-progress，当前fiber实例的alternate属性指向其work-in-progress，work-in-progress的alternate属性指向当前稳定fiber。
  const alternate = fiber.alternate;
  let queue1;
  let queue2;
  if (alternate === null) {
    // There's only one fiber.
    queue1 = fiber.updateQueue;
    queue2 = null;
    if (queue1 === null) {
      //如果当前组件没有等待setState的队列则创建一个，
        // 利用fiber当前已经记录并需要整合的state存储到queue1与fiber.updateQueue
      queue1 = fiber.updateQueue = createUpdateQueue(fiber.memoizedState);
    }
  } else {
    // There are two owners.
    // 如果fiber树以及workinprogress树都存在，下面的逻辑则会同步两个树的update队列
    queue1 = fiber.updateQueue;
    queue2 = alternate.updateQueue;
    // 当两个树的队列至少有一个不存在的时候执行队列创建或者复制操作
    if (queue1 === null) {
      if (queue2 === null) {
        // Neither fiber has an update queue. Create new ones.
        //  两个队列都没有则根据各自的memoizedState创建update队列
        queue1 = fiber.updateQueue = createUpdateQueue(fiber.memoizedState);
        queue2 = alternate.updateQueue = createUpdateQueue(
          alternate.memoizedState,
        );
      } else {
        // 如果有一个没有则复制另一个队列给它
        // Only one fiber has an update queue. Clone to create a new one.
        queue1 = fiber.updateQueue = cloneUpdateQueue(queue2);
      }
    } else {
      if (queue2 === null) {
        // 如果有一个没有则复制另一个队列给它
        // Only one fiber has an update queue. Clone to create a new one.
        queue2 = alternate.updateQueue = cloneUpdateQueue(queue1);
      } else {
        // Both owners have an update queue.
      }
    }
  }

  if (queue2 === null || queue1 === queue2) {
    // There's only a single queue.
    // 如果只有一个树，或者两棵树队列是同一个，则将传入的更新对象添加到第一个队列中
    appendUpdateToQueue(queue1, update);
  } else {
    // There are two queues. We need to append the update to both queues,
    // while accounting for the persistent structure of the list — we don't
    // want the same update to be added multiple times.
    //  如果两个队列存在，则将更新任务加入两个队列中，并避免被添加多次
    if (queue1.lastUpdate === null || queue2.lastUpdate === null) {
      // One of the queues is not empty. We must add the update to both queues.
      //  有一个队列不为空，将update添加到两个队列
      appendUpdateToQueue(queue1, update);
      appendUpdateToQueue(queue2, update);
    } else {
      // Both queues are non-empty. The last update is the same in both lists,
      // because of structural sharing. So, only append to one of the lists.
      appendUpdateToQueue(queue1, update);
      // But we still need to update the `lastUpdate` pointer of queue2.
      queue2.lastUpdate = update;
    }
  }
}

```

https://juejin.im/post/5d0106225188257d542e5dd1
https://juejin.im/post/5b87590051882542ff3e6c0b