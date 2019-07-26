## useRef

```
const refContainer = useRef(initialValue);
```

#### mount

mountRef 在组件挂载时获取了一个新的 hook，并将传入的初始值存入了一个对象的 current 属性中，这个对象被存入了 hook 的 memoizedState 属性中。并返回了一个 ref 对象。
```
function mountRef<T>(initialValue: T): {current: T} {
  const hook = mountWorkInProgressHook();
  const ref = {current: initialValue};
  if (__DEV__) {
    Object.seal(ref);
  }
  hook.memoizedState = ref;
  return ref;
}
```

#### update

获取了当前这个 hook，并直接读取返回了当前 hook 的缓存值。因此，使用 useRef 返回的 ref 对象在组件的整个生命周期内保持不变
```
function updateRef<T>(initialValue: T): {current: T} {
  const hook = updateWorkInProgressHook();
  return hook.memoizedState;
}
```

> 当 ref 对象内容发生变化时，useRef 并不会通知你。变更 .current 属性不会引发组件重新渲染。

