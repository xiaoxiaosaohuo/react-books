## useMemo

```
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);
```
#### mount

mountMemo 两个参数，一个 nextCreate 函数和 deps，它创建了一个 hook，并将 nextCreate 函数执行的返回值和 deps 作为数组存入 hook 的缓存中。

最后返回 nextCreate 函数执行的结果。

```
function mountMemo<T>(
  nextCreate: () => T,
  deps: Array<mixed> | void | null,
): T {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}
```

#### update

useMemo 在组件更新时会检查前后两次传入的 deps 是否按序相等。 如果相等，则返回之前存储的 nextCreate 的执行结果；如果不等，则会重新计算该值，并将最新的结果和新的 deps 进行缓存。最后返回这个新的计算结果。
```
function updateMemo<T>(
  nextCreate: () => T,
  deps: Array<mixed> | void | null,
): T {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;
  if (prevState !== null) {
    // Assume these are defined. If they're not, areHookInputsEqual will warn.
    if (nextDeps !== null) {
      const prevDeps: Array<mixed> | null = prevState[1];
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        return prevState[0];
      }
    }
  }
  const nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}
```

useCallback(fn, deps) 等同于 useMemo(() => fn, deps)

传递给useMemo的函数在渲染过程中运行。不要做那些在渲染时通常不会做的事情。例如，副作用属于useEffect，而不是useMemo。