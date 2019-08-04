## useReducer

useReducer和useState功能很像，当有复杂的状态逻辑涉及多个子值或下一个状态依赖于前一个状态的时候应该使用useReducer。
```
const [state, dispatch] = useReducer(reducer, initialArg, init);
```

- reducer reducer函数，处理状态更新
- initialArg 初始状态
- init 懒初始化函数


example

```
  function init(initialCount) {
  return {count: initialCount};
}

function reducer(state, action) {
  switch (action.type) {
    case 'increment':
      return {count: state.count + 1};
    case 'decrement':
      return {count: state.count - 1};
    case 'reset':
      return init(action.payload);
    default:
      throw new Error();
  }
}

function Counter({initialCount}) {
  const [state, dispatch] = useReducer(reducer, initialCount, init);
  return (
    <>
      Count: {state.count}
      <button
        onClick={() => dispatch({type: 'reset', payload: initialCount})}>

        Reset
      </button>
      <button onClick={() => dispatch({type: 'increment'})}>+</button>
      <button onClick={() => dispatch({type: 'decrement'})}>-</button>
    </>
  );
}

```

#### mount

```
function mountReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: I => S,
): [S, Dispatch<A>] {
  const hook = mountWorkInProgressHook();
  let initialState;
  if (init !== undefined) {
    initialState = init(initialArg);
  } else {
    initialState = ((initialArg: any): S);
  }
  hook.memoizedState = hook.baseState = initialState;
  const queue = (hook.queue = {
    last: null,
    dispatch: null,
    lastRenderedReducer: reducer,
    lastRenderedState: (initialState: any),
  });
  const dispatch: Dispatch<A> = (queue.dispatch = (dispatchAction.bind(
    null,
    // Flow doesn't know this is non-null, but we do.
    ((currentlyRenderingFiber: any): Fiber),
    queue,
  ): any));
  return [hook.memoizedState, dispatch];
}
```
#### update

update阶段调用updateReducer，前面介绍过



#### useReducer使用方法用例

[useReducer使用方法用例参考](https://github.com/xiaoxiaosaohuo/Note/issues/57)