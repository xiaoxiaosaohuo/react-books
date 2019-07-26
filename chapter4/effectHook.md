## Effect Hook

effect也就是在React中我们常说的side effect,Effect hook 和其他 hook 的行为有一些区别，在React中类似像componentDidMount，componentDidUpdate，componentWillUnmount生命周期方法中执行的副作用,常见的是我们希望在React更新DOM之后运行一些额外的代码。网络请求，手动DOM突变和日志记录等

现在effect hook共有
- useEffect
- useLayoutEffect

> useEffect会在每次更改作用于DOM并让浏览器绘制屏幕后去调用它。
> 每次重新渲染，都会生成新的effect，也就是说每个 effect “属于”一次特定的渲染



```
function useEffect(create, inputs) {
  var dispatcher = resolveDispatcher();
  return dispatcher.useEffect(create, inputs);
}
```
### useEffect--mount

```
function mountEffect(create, deps) {
  return mountEffectImpl(Update | Passive, UnmountPassive | MountPassive, create, deps);
}

function mountEffectImpl(fiberEffectTag, hookEffectTag, create, deps) {
  var hook = mountWorkInProgressHook();
  var nextDeps = deps === undefined ? null : deps;
  sideEffectTag |= fiberEffectTag;
  hook.memoizedState = pushEffect(hookEffectTag, create, undefined, nextDeps);
}


 function pushEffect(tag, create, destroy, deps) {
  var effect = {
    tag: tag,
    create: create,
    destroy: destroy,
    deps: deps,
    // Circular
    next: null
  };
  if (componentUpdateQueue === null) {
    componentUpdateQueue = createFunctionComponentUpdateQueue();
    componentUpdateQueue.lastEffect = effect.next = effect;
  } else {
    var _lastEffect = componentUpdateQueue.lastEffect;
    if (_lastEffect === null) {
      componentUpdateQueue.lastEffect = effect.next = effect;
    } else {
      var firstEffect = _lastEffect.next;
      _lastEffect.next = effect;
      effect.next = firstEffect;
      componentUpdateQueue.lastEffect = effect;
    }
  }
  return effect;
}

    
```
- mountEffectImpl使用 pushEffect 方法返回一个 Effect 对象，并将其存储到 hook 的缓存中。
- pushEffect 创建一个effect对象，添加effectTag，将其添加到componentUpdateQueue末尾，并返回这个对象
- 在renderWithHooks 方法中通过``` renderedWork.updateQueue = componentUpdateQueue;``` 将componentUpdateQueue更新到fiber的updateQueue，这个updateQueue会在commit阶段执行

### useEffect--update

```

function updateEffect(create, deps) {
  return updateEffectImpl(Update | Passive, UnmountPassive | MountPassive, create, deps);
}

function updateEffectImpl(fiberEffectTag, hookEffectTag, create, deps) {
  var hook = updateWorkInProgressHook();
  var nextDeps = deps === undefined ? null : deps;
  var destroy = undefined;

  if (currentHook !== null) {
      // 获取当前effect节点所在队列位置
    var prevEffect = currentHook.memoizedState;
    destroy = prevEffect.destroy;
    if (nextDeps !== null) {
      var prevDeps = prevEffect.deps;
      // 判断前后的 deps 是否相同
      if (areHookInputsEqual(nextDeps, prevDeps)) {
          //创建一个新的 Effect，并把它添加到更新队列
          //tag标记NoEffect$1 = 0
        pushEffect(NoEffect$1, create, destroy, nextDeps);
        return;
      }
    }
  }
  // 如果状态发生变化，则将当前effect的tag设置UnmountPassive | MountPassive，并后续在commitHookEffectList触发更新
  sideEffectTag |= fiberEffectTag;
  hook.memoizedState = pushEffect(hookEffectTag, create, destroy, nextDeps);
}
```

### updateLayoutEffect--mount

```
function mountLayoutEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null,
): void {
  return mountEffectImpl(
    UpdateEffect,
    UnmountMutation | MountLayout,
    create,
    deps,
  );
}
```

### updateLayoutEffect--update

```
function updateLayoutEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null,
): void {
  return updateEffectImpl(
    UpdateEffect,
    UnmountMutation | MountLayout,
    create,
    deps,
  );
}
```

可以看到updateLayoutEffect 的实现和 updateEffect 的实现一样。只是传入的tag不同。
fiberEffectTag和hookEffectTag会在commit阶段做对比，决定effect的执行时机。

### commit阶段Hook相关的内容

在commit阶段 有四个方法会执行hook相关的commit工作 ``` commitHookEffectList ```

- commitBeforeMutationLifeCycles
- commitPassiveHookEffects
- commitLifeCycles
- commitWork

commitHookEffectList 方法的核心工作

- 遍历effect链表，处理每个effect
- 根据传入的unmountTag和mountTag来判断是否需要执行对应的destory和create方法



```
function commitHookEffectList(
  unmountTag: number,
  mountTag: number,
  finishedWork: Fiber,
) {
  const updateQueue: FunctionComponentUpdateQueue | null = (finishedWork.updateQueue: any);
  let lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    do {
      if ((effect.tag & unmountTag) !== NoHookEffect) {
        // Unmount
        const destroy = effect.destroy;
        effect.destroy = undefined;
        if (destroy !== undefined) {
          destroy();
        }
      }
      if ((effect.tag & mountTag) !== NoHookEffect) {
        // Mount
        const create = effect.create;
        effect.destroy = create();
             
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```

- useLayoutEffect的create会在commitLifeCycles的时候被执行，destory会在commitWork的时候被执行，其执行过程跟componentDidMount和componentDidUpdate相似
- useEffect在commitRoot之后的commitPassiveHookEffects方法中执行。
  

*** 这里要说的是useEffect会进入异步调度流程 ***

在commitRoot有这样的一个判断

```
if (firstEffect !== null && rootWithPendingPassiveEffects !== null) {
  //这个commit包含passive effect，他们不需要执行直到下一次绘制之后，调度一个回调函数在一个异步事件中执行他们
    var callback = commitPassiveEffects.bind(null, root, firstEffect);
    if (enableSchedulerTracing) {
      
      callback = unstable_wrap(callback);
    }
    passiveEffectCallbackHandle = unstable_runWithPriority(unstable_NormalPriority, function () {
      return schedulePassiveEffects(callback);
    });
    passiveEffectCallback = callback;
  }
```


rootWithPendingPassiveEffects是在commitAllLifeCycles的时候如果发现更新中有passive effect的节点的话，就等于FiberRoot。

schedulePassiveEffects指向的是unstable_scheduleCallback（react-shceduler中的方法）

```
if (effectTag & Passive) {
      rootWithPendingPassiveEffects = finishedRoot;
  }

export function commitPassiveHookEffects(finishedWork: Fiber): void {
  commitHookEffectList(UnmountPassive, NoHookEffect, finishedWork);
  commitHookEffectList(NoHookEffect, MountPassive, finishedWork);
}
```

看到schedulePassiveEffects，可以想到这里就和react-scheduler有关系了，进入了异步调度流程。


所以 useEffect的destory和create都是异步调用的，React只会在浏览器绘制后运行effects。这使得你的应用更流畅因为大多数effects并不会阻塞屏幕的更新


## 如何正确的使用effect

请看 [React作者Dan关于effect的完全指南](https://overreacted.io/zh-hans/a-complete-guide-to-useeffect/)
