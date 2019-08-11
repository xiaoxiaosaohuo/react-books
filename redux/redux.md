<!--
 * @Description: In User Settings Edit
 * @Author: your name
 * @Date: 2019-08-10 12:47:53
 * @LastEditTime: 2019-08-10 14:54:25
 * @LastEditors: Please set LastEditors
 -->
## redux原理

现在单页面应用越来越复杂，我们必须管理很多不同的state,比如：服务响应数据，本地缓存数据，或者自己创建的数据，UI状态，路由，选择的tab,加载loading等等。

管理不断变化的状态是很难的一件事，如果一个model能更新另外一个model,一个view可以更新一个model,然后这个model更新其他的model,反过来会导致其他的view更新，最终导致无法理解发生了什么，很难定位或者复现产生问题的原因。根本原因是状态变得不可预测了。

redux的三个基本原则
- 单一的state
- state是只读的
- 改变由纯函数来执行


我们知道redux是一个管理状态的，状态就是数据，比如最简单的计数器
```
const state = {
    count:0
}
```
在redux中我们的state就类似于这种对象，但是redux在state发生了变化之后可以更新视图，而且还有更新state的操作。在redux中使用的是发布订阅模式.


redux对外暴露的API
- createStore,
- combineReducers,
- bindActionCreators,
- applyMiddleware,
- compose,

### createStore


自己来实现一个createStore

```
const createStore = function(preloadedState){
    let currentState = preloadedState;
    let currentListeners = [];
    let nextListeners = currentListeners;
    //获取状态
    function getState() {
        return currentState
    }
    //订阅
    function subscribe(listener) {
        if (typeof listener !== 'function') {
            throw new Error('Expected the listener to be a function.')
        }
        let isSubscribed = true;
        nextListeners.push(listener);
        return function unsubscribe() {
            if (!isSubscribed) {
                return
            }
            isSubscribed = false;
            const index = nextListeners.indexOf(listener)
            nextListeners.splice(index, 1)
            currentListeners = null
        }
    }
    function dispatch(newState) {
        currentState = newState;
        const listeners = (currentListeners = nextListeners)
            for (let i = 0; i < listeners.length; i++) {
            const listener = listeners[i]
            listener()
        }
    }
    return {
        dispatch,
        subscribe,
        getState,
    }
}

```

这种实现非常直观，可以看到dispatch传入的不是一个action,而是一个新对象

我们看用法

```
const initState = {
    count:0
}
const store = createStore(initState);
store.subscribe(()-=>{
    const state = store.getState();
    console.log(`${state.count}`)
})
store.dispatch({
    count:1
})

```

但是很明显我们可以随意的修改store,主要调用dispatch方法传入新对象即可，那么count可能被修改为任何值，这种修改没有任何约束肯定是不行的，否则很难排查问题。

我们需要一个修改store的动作，这个动作只能修改特定的state,在redux中有reducer和action这两种概念。action是修改的动作，reducer是处理该动作的一个方法，会返回新的state，reducer必须是纯函数。

定义一个reducer

```
const initState = {
    count:0
}
function countReuder(state = initState,action){
    switch(action.type){
        case 'INCREMENT':
            reutrn {...state,count:state.count+1}
        case 'DECREMENT':
            reutrn {...state,count:state.count-1}
        default:
            return state;
    }
}
```
我们需要把这个修改state的方法告诉store,在调用store.dispatch后要执行这个方法。


```
const createStore = function(reducer,preloadedState){
    let currentState = preloadedState;
    let currentListeners = [];
    let nextListeners = currentListeners;
    //获取状态
    function getState() {
        return currentState
    }
    //订阅
    function subscribe(listener) {
        if (typeof listener !== 'function') {
            throw new Error('Expected the listener to be a function.')
        }
        let isSubscribed = true;
        nextListeners.push(listener);
        return function unsubscribe() {
            if (!isSubscribed) {
                return
            }
            isSubscribed = false;
            const index = nextListeners.indexOf(listener)
            nextListeners.splice(index, 1)
            currentListeners = null
        }
    }
    function dispatch(action) {
        
        currentState = reducer(action);
        const listeners = (currentListeners = nextListeners)
            for (let i = 0; i < listeners.length; i++) {
            const listener = listeners[i]
            listener()
        }
    }
    return {
        dispatch,
        subscribe,
        getState,
    }
}
```

#### reducer的拆分和合并

在实际应用中，我们会按照职责或者功能的不同拆分出不同的reducer，不可能一个app就一个reducer，那得多么庞大啊，所以redux提供了combineReducers的方法，将多个reducer合并为一个。

combineReducers 接受多个reducer,返回一个新的reducer。
```
function combineReducers(reducers){
    const reducersKeys = Object.keys(reducers);
    const finalReducers = {};
    for (let i = 0; i < reducersKeys.length; i++){
        const key = reducerKeys[i];
        if (typeof reducers[key] === 'function') {
            finalReducers[key] = reducers[key]
        }
    }

    const finalReducerKeys = Object.keys(finalReducers)
    //返回新的reducer
    function combination (state={},action){
        let hasChanged = false;
        const nextState = {};
        for (let i = 0; i < finalReducerKeys.length; i++) {
            const key = finalReducerKeys[i];
            const reducer = finalReducers[key];
            const previousStateForKey = state[key];
            const nextStateForKey = reducer(previousStateForKey, action);
            nextState[key] = nextStateForKey;
            hasChanged = hasChanged || nextStateForKey !== previousStateForKey
        }
        return hasChanged ? nextState : state

    }

}
```


### state初始化

上面代码有个问题，在createStore的时候，我们的state是`undefined`

我们需要在初始化的时候主动的dispatch一个动作，让每个reducer返回default中的state，这样就能获得正确的初始化的state了。


```
const randomString = () =>
  Math.random()
    .toString(36)
    .substring(7)
    .split('')
    .join('.')

const ActionTypes = {
  INIT: `@@redux/INIT${randomString()}`,
  REPLACE: `@@redux/REPLACE${randomString()}`,
  PROBE_UNKNOWN_ACTION: () => `@@redux/PROBE_UNKNOWN_ACTION${randomString()}`
}


const createStore = function(reducer,preloadedState){
    let currentState = preloadedState;
    let currentListeners = [];
    let nextListeners = currentListeners;
    //获取状态
    function getState() {
        return currentState
    }
    //订阅
    function subscribe(listener) {
        if (typeof listener !== 'function') {
            throw new Error('Expected the listener to be a function.')
        }
        let isSubscribed = true;
        nextListeners.push(listener);
        return function unsubscribe() {
            if (!isSubscribed) {
                return
            }
            isSubscribed = false;
            const index = nextListeners.indexOf(listener)
            nextListeners.splice(index, 1)
            currentListeners = null
        }
    }
    function dispatch(action) {
        
        currentState = reducer(action);
        const listeners = (currentListeners = nextListeners)
            for (let i = 0; i < listeners.length; i++) {
            const listener = listeners[i]
            listener()
        }
        return action
    }
    dispatch({ type: ActionTypes.INIT })
    return {
        dispatch,
        subscribe,
        getState,
    }
}
```

### 中间件

redux有强大的中间件模式，方便扩展功能。

比如我们想打印日志

```
const store = createStore(reducer);
const next = store.dispatch;

/*重写store.dispatch*/
store.dispatch = (action) => {
    console.log('prevState', store.getState());
    console.log('action', action);
    next(action);
    console.log('nextState', store.getState());
}
```


假如有多个中间件这中方式就不灵活了，没法扩展,比如记录错误的中间件。

```
store.dispatch = (action) => {
  try {
    console.log('prevState', store.getState());
    console.log('action', action);
    next(action);
    console.log('nextState', store.getState());
  } catch (err) {
    console.error('错误报告: ', err)
  }
}
```

- 首先把中间提出来
```
const loggerMiddleware = (action)=>{
    console.log('prevState', store.getState());
    console.log('action', action);
    next(action);
    console.log('nextState', store.getState());
}
const errorMiddleware = (action)=>{
    try {
    loggerMiddleware(action);
  } catch (err) {
    console.error('错误报告: ', err)
  }
}
store.dispatch = errorMiddleware;
```

- 从errorMiddleware中提出loggerMiddleware，当做参数传入
  
```
const store = createStore(reducer);
const next = store.dispatch;
const loggerMiddleware = (next)=>(action)=>{
    console.log('prevState', store.getState());
    console.log('action', action);
    next(action);
    console.log('nextState', store.getState());
}
const errorMiddleware = (next)=>(action)=>{
    try {
    next(action);
  } catch (err) {
    console.error('错误报告: ', err)
  }
}
store.dispatch = errorMiddleware(loggerMiddleware(next));
```

- 抽取中间件中的store

```
const store = createStore(reducer);
const next = store.dispatch;
const loggerMiddleware = (store)=>(next)=>(action)=>{
    console.log('prevState', store.getState());
    console.log('action', action);
    next(action);
    console.log('nextState', store.getState());
}
const errorMiddleware = (store)=>(next)=>(action)=>{
    try {
    next(action);
  } catch (err) {
    console.error('错误报告: ', err)
  }
}
const logger = loggerMiddleware(store);
const error  = errorMiddleware(store);
store.dispatch = error(logger(next));
```

在redux中提供了applyMiddleware方法，极大的方便中间的使用。


```
const store = createStore(
		reducer,
		initialState,
		applyMiddleware(loggerMiddleware,errorMiddleware)
)
```

### applyMiddleware

```
//createStore 
if (typeof enhancer !== 'undefined') {
    if (typeof enhancer !== 'function') {
      throw new Error('Expected the enhancer to be a function.')
    }

    return enhancer(createStore)(reducer, preloadedState)
  }


//applyMiddleware

export default function applyMiddleware(...middlewares) {
  return createStore => (...args) => {
      //生成store
    const store = createStore(...args)
    let dispatch = () => {
      throw new Error(
        'Dispatching while constructing your middleware is not allowed. ' +
          'Other middleware would not be applied to this dispatch.'
      )
    }

    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args)
    }
    //给每个middleware传入store的getState和dispatch方法
    //chain相当于[logger,error];
    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    //执行中间件返回dispatch 相当于error(logger(store.dispatch))
    dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}
export default function compose(...funcs) {
  if (funcs.length === 0) {
    return arg => arg
  }

  if (funcs.length === 1) {
    return funcs[0]
  }

  return funcs.reduce((a, b) => (...args) => a(b(...args)))
}
```
在redux的createStore方法中一个判断，如果enhancer存在，就调用.



```
return enhancer(createStore)(reducer, preloadedState)
```

enhancer就是applyMiddleware返回的一个函数，该函数接受旧的createStore的为参数。

### bindActionCreators

过闭包，把 dispatch 和 actionCreator 隐藏起来，让其他地方感知不到 redux 的存在。
```
function bindActionCreator(actionCreator, dispatch) {
  return function() {
    return dispatch(actionCreator.apply(this, arguments))
  }
}
export default function bindActionCreators(actionCreators, dispatch) {
  if (typeof actionCreators === 'function') {
    return bindActionCreator(actionCreators, dispatch)
  }

  const boundActionCreators = {}
  for (const key in actionCreators) {
    const actionCreator = actionCreators[key]
    if (typeof actionCreator === 'function') {
      boundActionCreators[key] = bindActionCreator(actionCreator, dispatch)
    }
  }
  return boundActionCreators
}
```