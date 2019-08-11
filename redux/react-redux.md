<!--
 * @Description: In User Settings Edit
 * @Author: jinxin
 * @Date: 2019-08-11 12:35:17
 * @LastEditTime: 2019-08-12 01:13:32
 * @LastEditors: Please set LastEditors
 -->
## React-Redux原理

React是一个UI库，Redux是一个状态管理库，React-Redux的角色就是连接UI和store的中间层，是MVVM中的VM。


## UI更新逻辑流程

直接使用Redux更新UI通常需要以下几个步骤

1. 创建一个Redux store
2. 订阅更新
3. 在订阅回调中，
    - 获取当前的store state
    - 抽取UI需要的数据
    - 用数据更新UI
4. 用初始状态渲染UI
5. 通过dispatch Redux actions来响应UI操作

```
// 1) Create a store
const store = createStore(counter)

// 2) Subscribe to store updates
store.subscribe(render);

const valueEl = document.getElementById('value');

// 3. When the subscription callback runs:
function render() {
    // 3.1) Get the current store state
    const state = store.getState();

    // 3.2) Extract the data you want
    const newValue = state.toString();

    // 3.3) Update the UI with the new value
    valueEl.innerHTML = newValue;
}

// 4) Display the UI with the initial store state
render();

// 5) Dispatch actions based on UI inputs
document.getElementById("increment")
    .addEventListener('click', () => {
        store.dispatch({type : "INCREMENT"});
    })
```

任何整合Redux的UI层都需要这几个步骤。

你会发现，处理订阅逻辑，检查数据是否更新，触发重新render等逻辑是可以复用的。这就是React-Redux的作用


在React-Redux中负责这些逻辑的是一个叫connect的函数，下面是一个简化版或者演示版的connect,不是真正的实现，但是有助于直观的理解connect。

它的主要功能是注入redux相关的属性到组件中。

```
function connect(mapStateToProps, mapDispatchToProps) {
  return function (WrappedComponent) {
    // 返回一个组件
    return class extends React.Component {
      render() {
        return (
          //渲染传入的组件
          <WrappedComponent
            {...this.props}
            {/* 从redux store中计算的新属性*/}
            {...mapStateToProps(store.getState(), this.props)}
            {...mapDispatchToProps(store.dispatch, this.props)}
          />
        )
      }
      
      componentDidMount() {
        // 订阅
        this.unsubscribe = store.subscribe(this.handleChange.bind(this))
      }
      
      componentWillUnmount() {
        // 取消订阅
        this.unsubscribe()
      }
    
      handleChange() {
        // 更新
        this.forceUpdate()
      }
    }
  }
}
```

connect API 的关键几点
- connect是一个函数，返回一个函数，该函数返回一个组件包裹器
- 被包裹组件的属性是包裹器组件属性和redux中相关属性的合并项
- 每一个包裹器组件都是Redux store的独立的一个订阅者
- 包裹器组件抽离了订阅，取消订阅，更新UI，性能优化等逻辑
- 作为使用者只要要声明从store从获取什么数据，和修改store的逻辑即可
  

## React-Redux 源码分析

React-Redux原理虽然听起来简单，但是React-Redux的源码其实有点复杂。

#### 工具方法---Subscribtion

```
import { getBatch } from './batch'


const CLEARED = null
const nullListeners = { notify() {} }

function createListenerCollection() {
  const batch = getBatch()
  // the current/next pattern is copied from redux's createStore code.
  // TODO: refactor+expose that code to be reusable here?
  let current = []
  let next = []

  return {
    clear() {
      next = CLEARED
      current = CLEARED
    },

    notify() {
      const listeners = (current = next)
      batch(() => {
        for (let i = 0; i < listeners.length; i++) {
          listeners[i]()
        }
      })
    },

    get() {
      return next
    },

    subscribe(listener) {
      let isSubscribed = true
      if (next === current) next = current.slice()
      next.push(listener)

      return function unsubscribe() {
        if (!isSubscribed || current === CLEARED) return
        isSubscribed = false

        if (next === current) next = current.slice()
        next.splice(next.indexOf(listener), 1)
      }
    }
  }
}

export default class Subscription {
  constructor(store, parentSub) {
    this.store = store
    this.parentSub = parentSub
    this.unsubscribe = null
    this.listeners = nullListeners

    this.handleChangeWrapper = this.handleChangeWrapper.bind(this)
  }

  addNestedSub(listener) {
    this.trySubscribe()
    return this.listeners.subscribe(listener)
  }

  notifyNestedSubs() {
    this.listeners.notify()
  }

  handleChangeWrapper() {
    if (this.onStateChange) {
      this.onStateChange()
    }
  }

  isSubscribed() {
    return Boolean(this.unsubscribe)
  }

  trySubscribe() {
    if (!this.unsubscribe) {
      this.unsubscribe = this.parentSub
        ? this.parentSub.addNestedSub(this.handleChangeWrapper)
        : this.store.subscribe(this.handleChangeWrapper)

      this.listeners = createListenerCollection()
    }
  }

  tryUnsubscribe() {
    if (this.unsubscribe) {
      this.unsubscribe()
      this.unsubscribe = null
      this.listeners.clear()
      this.listeners = nullListeners
    }
  }
}

```

// encapsulates the subscription logic for connecting a component to the redux store, as
// well as nesting subscriptions of descendant components, so that we can ensure the
// ancestor components re-render before descendants

- createListenerCollection 实现订阅模式，共暴露4个方法
    - clear 清空监听器
    - notify 通知
    - get 获取当前监听器集合的getter
    - subscribe 订阅方法，用于添加监听器

- Subscription类，主要用于处理订阅逻辑
    - 封装了连接组件到redux store的订阅逻辑
    - 确保自上而下的订阅逻辑，在connect方法中，会将store的实例和subscription实例放进context中，如果subscription不存在，就直接订阅store,否则订阅subscription，这意味着每个连接的组件都有效地订阅其最近的连接祖先

### Provider

```

import { ReactReduxContext } from './Context'
import Subscription from '../utils/Subscription'

class Provider extends React.Component {
  constructor(props) {
    super(props)
    const { store } = props

    this.notifySubscribers = this.notifySubscribers.bind(this)
    const subscription = new Subscription(store)
    subscription.onStateChange = this.notifySubscribers

    this.state = {
      store,
      subscription
    }

    this.previousState = store.getState()
  }

  componentDidMount() {
    this.state.subscription.trySubscribe()

    if (this.previousState !== this.props.store.getState()) {
      this.state.subscription.notifyNestedSubs()
    }
  }

  componentWillUnmount() {
    if (this.unsubscribe) this.unsubscribe()

    this.state.subscription.tryUnsubscribe()
  }

  componentDidUpdate(prevProps) {
    if (this.props.store !== prevProps.store) {
      this.state.subscription.tryUnsubscribe()
      const subscription = new Subscription(this.props.store)
      subscription.onStateChange = this.notifySubscribers
      this.setState({ store: this.props.store, subscription })
    }
  }

  notifySubscribers() {
    this.state.subscription.notifyNestedSubs()
  }

  render() {
    const Context = this.props.context || ReactReduxContext

    return (
      <Context.Provider value={this.state}>
        {this.props.children}
      </Context.Provider>
    )
  }
}



export default Provider

```
- Provider 代码比较简单，注意value是store和subscription

- onStateChange 方法最终会调用subscription的notifyNestedSubs方法

### connect 方法

```

function match(arg, factories, name) {
  for (let i = factories.length - 1; i >= 0; i--) {
    const result = factories[i](arg)
    if (result) return result
  }

  return (dispatch, options) => {
    throw new Error(
      `Invalid value of type ${typeof arg} for ${name} argument when connecting component ${
        options.wrappedComponentName
      }.`
    )
  }
}

export function createConnect({
  connectHOC = connectAdvanced,
  mapStateToPropsFactories = defaultMapStateToPropsFactories,
  mapDispatchToPropsFactories = defaultMapDispatchToPropsFactories,
  mergePropsFactories = defaultMergePropsFactories,
  selectorFactory = defaultSelectorFactory
} = {}) {
  return function connect(
    mapStateToProps,
    mapDispatchToProps,
    mergeProps,
    {
      pure = true,
      areStatesEqual = strictEqual,
      areOwnPropsEqual = shallowEqual,
      areStatePropsEqual = shallowEqual,
      areMergedPropsEqual = shallowEqual,
      ...extraOptions
    } = {}
  ) {
    const initMapStateToProps = match(
      mapStateToProps,
      mapStateToPropsFactories,
      'mapStateToProps'
    )
    const initMapDispatchToProps = match(
      mapDispatchToProps,
      mapDispatchToPropsFactories,
      'mapDispatchToProps'
    )
    const initMergeProps = match(mergeProps, mergePropsFactories, 'mergeProps')

    return connectHOC(selectorFactory, {
      // used in error messages
      methodName: 'connect',

      // used to compute Connect's displayName from the wrapped component's displayName.
      getDisplayName: name => `Connect(${name})`,

      // if mapStateToProps is falsy, the Connect component doesn't subscribe to store state changes
      shouldHandleStateChanges: Boolean(mapStateToProps),

      // 以下参数将传递给 selectorFactory
      initMapStateToProps,
      initMapDispatchToProps,
      initMergeProps,
      pure,
      areStatesEqual,
      areOwnPropsEqual,
      areStatePropsEqual,
      areMergedPropsEqual,

      // any extra options args can override defaults of connect or connectAdvanced
      ...extraOptions
    })
  }
}

export default createConnect()
```
connect 方法其实是connectAdvanced方法的门面模式实现，返回一个函数接受我们自定义的
`mapStateToProps`, `mapDispatchToProps`,`mergeProps`参数，对传入的参数做一些处理，然后调用了connectAdvanced，真正的订阅，更新等逻辑是在connectAdvanced中完成的。
  
  

##### mapStateToPropsFactories，以initMapStateToProps为例
```
const initMapStateToProps = match(
    mapStateToProps,
    mapStateToPropsFactories,
    'mapStateToProps'
)

import { wrapMapToPropsConstant, wrapMapToPropsFunc } from './wrapMapToProps'

export function whenMapStateToPropsIsFunction(mapStateToProps) {
  return typeof mapStateToProps === 'function'
    ? wrapMapToPropsFunc(mapStateToProps, 'mapStateToProps')
    : undefined
}

export function whenMapStateToPropsIsMissing(mapStateToProps) {
  return !mapStateToProps ? wrapMapToPropsConstant(() => ({})) : undefined
}

export default [whenMapStateToPropsIsFunction, whenMapStateToPropsIsMissing]

```
mapStateToPropsFactories 是一个数组，每一项都对参数进行归一化处理，比如`whenMapStateToPropsIsMissing`检查传入的`mapStateToProps`是否存在，存在就返回undefiend ,接下来交给`whenMapStateToPropsIsFunction`处理，用`wrapMapToPropsFunc`对`mapStateToProps`进一步处理。


#### wrapMapToPropsFunc

```
export function wrapMapToPropsFunc(mapToProps, methodName) {
  return function initProxySelector(dispatch, { displayName }) {
    const proxy = function mapToPropsProxy(stateOrDispatch, ownProps) {
      return proxy.dependsOnOwnProps
        ? proxy.mapToProps(stateOrDispatch, ownProps)
        : proxy.mapToProps(stateOrDispatch)
    }

    // allow detectFactoryAndVerify to get ownProps
    proxy.dependsOnOwnProps = true

    proxy.mapToProps = function detectFactoryAndVerify(
      stateOrDispatch,
      ownProps
    ) {
      proxy.mapToProps = mapToProps
      proxy.dependsOnOwnProps = getDependsOnOwnProps(mapToProps)
      let props = proxy(stateOrDispatch, ownProps)

      if (typeof props === 'function') {
        proxy.mapToProps = props
        proxy.dependsOnOwnProps = getDependsOnOwnProps(props)
        props = proxy(stateOrDispatch, ownProps)
      }

      if (process.env.NODE_ENV !== 'production')
        verifyPlainObject(props, displayName, methodName)

      return props
    }

    return proxy
  }
}
```

wrapMapToPropsFunc 返回一个代理选择器`initProxySelector` ,通过闭包，该方法保存了mapStateToProps方法。

`initProxySelector` 接受`dispatch`后返回 proxy。

*** proxy主要功能是侦测mapToProps方法的调用是否依赖props(父级组件传递下来的props) ***

- 第一次调用proxy，proxy.mapToProps是`detectFactoryAndVerify`,会调用getDependsOnOwnProps来判断mapToProps是否依赖props，重置dependsOnOwnProps,mapToProps。
-  第二次调用proxy，proxy.mapToProps是我们传入的mapToProps，判断返回的是函数还是普通对象。最终返回props。


#### selectorFactory

selectorFactory中的方法分为pure和impure两大类
如果是pure的话，会缓存计算结果，如果属性比较无变化，就直接返回上次的计算结果，无特殊情况，我们都是调用的pure方法。



```
import verifySubselectors from './verifySubselectors'

export function impureFinalPropsSelectorFactory(
  mapStateToProps,
  mapDispatchToProps,
  mergeProps,
  dispatch
) {
  return function impureFinalPropsSelector(state, ownProps) {
    return mergeProps(
      mapStateToProps(state, ownProps),
      mapDispatchToProps(dispatch, ownProps),
      ownProps
    )
  }
}

export function pureFinalPropsSelectorFactory(
  mapStateToProps,
  mapDispatchToProps,
  mergeProps,
  dispatch,
  { areStatesEqual, areOwnPropsEqual, areStatePropsEqual }
) {
  //标记是否第一次调用
  let hasRunAtLeastOnce = false
  let state
  let ownProps
  let stateProps
  let dispatchProps
  let mergedProps

  function handleFirstCall(firstState, firstOwnProps) {
    state = firstState
    ownProps = firstOwnProps
    stateProps = mapStateToProps(state, ownProps)
    dispatchProps = mapDispatchToProps(dispatch, ownProps)
    mergedProps = mergeProps(stateProps, dispatchProps, ownProps)
    hasRunAtLeastOnce = true
    return mergedProps
  }

  function handleNewPropsAndNewState() {
    stateProps = mapStateToProps(state, ownProps)

    if (mapDispatchToProps.dependsOnOwnProps)
      dispatchProps = mapDispatchToProps(dispatch, ownProps)

    mergedProps = mergeProps(stateProps, dispatchProps, ownProps)
    return mergedProps
  }

  function handleNewProps() {
    if (mapStateToProps.dependsOnOwnProps)
      stateProps = mapStateToProps(state, ownProps)

    if (mapDispatchToProps.dependsOnOwnProps)
      dispatchProps = mapDispatchToProps(dispatch, ownProps)

    mergedProps = mergeProps(stateProps, dispatchProps, ownProps)
    return mergedProps
  }

  function handleNewState() {
    const nextStateProps = mapStateToProps(state, ownProps)
    const statePropsChanged = !areStatePropsEqual(nextStateProps, stateProps)
    stateProps = nextStateProps
  //mapStateToProps 计算出来的结果无变化就直接返回上次的
    if (statePropsChanged)
      mergedProps = mergeProps(stateProps, dispatchProps, ownProps)

    return mergedProps
  }

  function handleSubsequentCalls(nextState, nextOwnProps) {
    const propsChanged = !areOwnPropsEqual(nextOwnProps, ownProps)
    const stateChanged = !areStatesEqual(nextState, state)
    state = nextState
    ownProps = nextOwnProps

    if (propsChanged && stateChanged) return handleNewPropsAndNewState()
    if (propsChanged) return handleNewProps()
    if (stateChanged) return handleNewState()
    return mergedProps
  }

  return function pureFinalPropsSelector(nextState, nextOwnProps) {
    return hasRunAtLeastOnce
      ? handleSubsequentCalls(nextState, nextOwnProps)
      : handleFirstCall(nextState, nextOwnProps)
  }
}


export default function finalPropsSelectorFactory(
  dispatch,
  { initMapStateToProps, initMapDispatchToProps, initMergeProps, ...options }
) {
  const mapStateToProps = initMapStateToProps(dispatch, options)
  const mapDispatchToProps = initMapDispatchToProps(dispatch, options)
  const mergeProps = initMergeProps(dispatch, options)

  if (process.env.NODE_ENV !== 'production') {
    verifySubselectors(
      mapStateToProps,
      mapDispatchToProps,
      mergeProps,
      options.displayName
    )
  }

  const selectorFactory = options.pure
    ? pureFinalPropsSelectorFactory
    : impureFinalPropsSelectorFactory

  return selectorFactory(
    mapStateToProps,
    mapDispatchToProps,
    mergeProps,
    dispatch,
    options
  )
}

```

- `finalPropsSelectorFactory` 调用`initMapStateToProps`等方法返回`mapStateToProps`和 `mapDispatchToProps` 的proxy方法，用于后续侦测是否依赖props
- 根据pure的值决定调用 `pureFinalPropsSelectorFactory `还是 `impureFinalPropsSelectorFactory` 这两个方法之一就是selectorFactory
- selectorFactory方法返回最终的FinalPropsSelector作为选择器，其签名是`（nextState, nextOwnProps)=>nextFinalProps` 。

从以上分析来看 connect把参数传递给connectAdvanced，然后经过selectorFactory处理返回最终的selector。其调用过程类似：

```(dispatch, options) => (nextState, nextOwnProps) => nextFinalProps ```

nextFinalProps 就是最终计算给组件使用的属性。


selectorFactory 的作用就是从以下方法或者参数中返回最终的selector。
  ```
  mapStateToProps,
  mapStateToPropsFactories, 
  mapDispatchToProps, 
  mapDispatchToPropsFactories,
  mergeProps,
  mergePropsFactories, 
  pure
  ```
这个selector用于从 state，props,dispatch中计算组件需要的新props。




## connectAdvanced

```
import hoistStatics from 'hoist-non-react-statics'
import invariant from 'invariant'

import { isValidElementType, isContextConsumer } from 'react-is'
import Subscription from '../utils/Subscription'

import { ReactReduxContext } from './Context'

// Define some constant arrays just to avoid re-creating these
const EMPTY_ARRAY = []
const NO_SUBSCRIPTION_ARRAY = [null, null]

const stringifyComponent = Comp => {
  try {
    return JSON.stringify(Comp)
  } catch (err) {
    return String(Comp)
  }
}

//该reducer用于在store更新的时候去更新组件
function storeStateUpdatesReducer(state, action) {
  const [, updateCount] = state
  return [action.payload, updateCount + 1]
}

const initStateUpdates = () => [null, 0]


//在server中使用React.useEffect
//在浏览器中使用useLayoutEffect ，useLayoutEffec是同步的且在commit dom的时候会调用，方便及时的通过ref保存最新的props.

const useIsomorphicLayoutEffect =
  typeof window !== 'undefined' &&
  typeof window.document !== 'undefined' &&
  typeof window.document.createElement !== 'undefined'
    ? React.useLayoutEffect
    : React.useEffect

export default function connectAdvanced(
  selectorFactory,
  {
    //这个方法用于计算HOC的displayName
    getDisplayName = name => `ConnectAdvanced(${name})`,

    methodName = 'connectAdvanced',

    // 统计渲染次数
    renderCountProp = undefined,

    //判定HOC是否订阅store的变化
    shouldHandleStateChanges = true,

    storeKey = 'store',

    withRef = false,

    forwardRef = false,

    context = ReactReduxContext,

    ...connectOptions
  } = {}
) {

  const Context = context

  return function wrapWithConnect(WrappedComponent) {

    const wrappedComponentName =
      WrappedComponent.displayName || WrappedComponent.name || 'Component'

    const displayName = getDisplayName(wrappedComponentName)

    const selectorFactoryOptions = {
      ...connectOptions,
      getDisplayName,
      methodName,
      renderCountProp,
      shouldHandleStateChanges,
      storeKey,
      displayName,
      wrappedComponentName,
      WrappedComponent
    }

    const { pure } = connectOptions

   //从selectorFactory 返回的finalSelector
    function createChildSelector(store) {

      return selectorFactory(store.dispatch, selectorFactoryOptions)
    }

    // pure 的话就使用React.useMemo进行缓存，否则会直调用
    const usePureOnlyMemo = pure ? React.useMemo : callback => callback()

    function ConnectFunction(props) {

      const [propsContext, forwardedRef, wrapperProps] = React.useMemo(() => {
        //挑出forwardedRef属性，并对所有属性缓存
        const { forwardedRef, ...wrapperProps } = props
        return [props.context, forwardedRef, wrapperProps]
      }, [props])

    // 检查是使用用户传入的context还是 ReactReduxContext，并缓存检查的结果
      const ContextToUse = React.useMemo(() => {
        return propsContext &&
          propsContext.Consumer &&
          isContextConsumer(<propsContext.Consumer />)
          ? propsContext
          : Context
      }, [propsContext, Context])

      // 获取 store 和 祖先subscription
      const contextValue = React.useContext(ContextToUse)

      // 检查store来源
      const didStoreComeFromProps = Boolean(props.store)
      const didStoreComeFromContext =
        Boolean(contextValue) && Boolean(contextValue.store)

      const store = props.store || contextValue.store

      const childPropsSelector = React.useMemo(() => {
        //store发生变化就重新创建selector
        return createChildSelector(store)
      }, [store])

      const [subscription, notifyNestedSubs] = React.useMemo(() => {
        if (!shouldHandleStateChanges) return NO_SUBSCRIPTION_ARRAY
        //Subscription来源需要匹配store的来源，比如props,context
        //一个组件通过props connect到store时不应该使用来自context的subscription，反之亦然。

        // 第二个参数是parentSub
        const subscription = new Subscription(
          store,
          didStoreComeFromProps ? null : contextValue.subscription
        )
        
        const notifyNestedSubs = subscription.notifyNestedSubs.bind(
          subscription
        )

        return [subscription, notifyNestedSubs]
      }, [store, didStoreComeFromProps, contextValue])

      // 决定将哪个{store, subscription}组合传递到嵌套的context中，并且缓存之
      const overriddenContextValue = React.useMemo(() => {
        if (didStoreComeFromProps) {
          //该组件直接订阅来自props的store
          // 我们并不想后代组件从这个store中读取值，所以传递最近的祖先的contextValue
          return contextValue
        }

        // 将该组件的subscription实例放进context，那么被connected的后代就不会更新知道该组件更新完毕，完美符合React的top-down数据流设计。
        return {
          ...contextValue,
          subscription
        }
      }, [didStoreComeFromProps, contextValue, subscription])

      // 当Redux的store变化的时候，强制组件渲染
      const [
        [previousStateUpdateResult],
        forceComponentUpdateDispatch
      ] = React.useReducer(storeStateUpdatesReducer, EMPTY_ARRAY, initStateUpdates)

      // 处理错误
      if (previousStateUpdateResult && previousStateUpdateResult.error) {
        throw previousStateUpdateResult.error
      }

      // 设置refs
      const lastChildProps = React.useRef()
      const lastWrapperProps = React.useRef(wrapperProps)
      const childPropsFromStoreUpdate = React.useRef()
      const renderIsScheduled = React.useRef(false)

      const actualChildProps = usePureOnlyMemo(() => {
        if (
          childPropsFromStoreUpdate.current &&
          wrapperProps === lastWrapperProps.current
        ) {
          return childPropsFromStoreUpdate.current
        }
      //计算新的props
        return childPropsSelector(store.getState(), wrapperProps)
      }, [store, previousStateUpdateResult, wrapperProps])

      useIsomorphicLayoutEffect(() => {
        // 同步更新refs
        lastWrapperProps.current = wrapperProps
        lastChildProps.current = actualChildProps
        renderIsScheduled.current = false

        if (childPropsFromStoreUpdate.current) {
          childPropsFromStoreUpdate.current = null
          notifyNestedSubs()
        }
      })

      useIsomorphicLayoutEffect(() => {
        // 没有订阅store，就直接返回
        if (!shouldHandleStateChanges) return

        // 检查是否卸载
        let didUnsubscribe = false
        let lastThrownError = null
        //当store subscription 更新传播到该组件就会运行checkForUpdates
        const checkForUpdates = () => {
          if (didUnsubscribe) {
            return
          }

          const latestStoreState = store.getState()

          let newChildProps, error
          try {
            //计算新props
            newChildProps = childPropsSelector(
              latestStoreState,
              lastWrapperProps.current
            )
          } catch (e) {
            error = e
            lastThrownError = e
          }

          if (!error) {
            lastThrownError = null
          }

          if (newChildProps === lastChildProps.current) {
            if (!renderIsScheduled.current) {
              notifyNestedSubs()
            }
          } else {
            // 保存 新props到refs 
            lastChildProps.current = newChildProps
            childPropsFromStoreUpdate.current = newChildProps
            renderIsScheduled.current = true
            //强制渲染
            forceComponentUpdateDispatch({
              type: 'STORE_UPDATED',
              payload: {
                latestStoreState,
                error
              }
            })
          }
        }
        //订阅最近的被connected 的祖先
        subscription.onStateChange = checkForUpdates;
        //上面实例化subscription的时候传递了parentSub,这里trySubscribe会调用parentSub的addNestedSub方法，从而在父级的listener中添加了checkForUpdates作为监听器
        subscription.trySubscribe()

        // 第一render之后主动调用一次。
        checkForUpdates()

        const unsubscribeWrapper = () => {
          didUnsubscribe = true
          subscription.tryUnsubscribe()

          if (lastThrownError) {
            throw lastThrownError
          }
        }

        return unsubscribeWrapper
      }, [store, subscription, childPropsSelector])

      // 渲染被包裹的组件，并缓存
      const renderedWrappedComponent = React.useMemo(
        () => <WrappedComponent {...actualChildProps} ref={forwardedRef} />,
        [forwardedRef, WrappedComponent, actualChildProps]
      )

  
      //如果React检查到组件无变化，就会bails out，不会对子组件进行重复渲染。
      const renderedChild = React.useMemo(() => {
        if (shouldHandleStateChanges) {
          //如果该组件订阅了store的更新，需要将subscription实例传递给其后代，意味着将渲染同一个Context 实例,但是其value不同
          return (
            <ContextToUse.Provider value={overriddenContextValue}>
              {renderedWrappedComponent}
            </ContextToUse.Provider>
          )
        }

        return renderedWrappedComponent
      }, [ContextToUse, renderedWrappedComponent, overriddenContextValue])

      return renderedChild
    }

    const Connect = pure ? React.memo(ConnectFunction) : ConnectFunction

    Connect.WrappedComponent = WrappedComponent
    Connect.displayName = displayName

    if (forwardRef) {
      const forwarded = React.forwardRef(function forwardConnectRef(
        props,
        ref
      ) {
        return <Connect {...props} forwardedRef={ref} />
      })

      forwarded.displayName = displayName
      forwarded.WrappedComponent = WrappedComponent
      return hoistStatics(forwarded, WrappedComponent)
    }

    return hoistStatics(Connect, WrappedComponent)
  }
}

```