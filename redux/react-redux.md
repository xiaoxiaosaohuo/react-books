<!--
 * @Description: In User Settings Edit
 * @Author: jinxin
 * @Date: 2019-08-11 12:35:17
 * @LastEditTime: 2019-08-11 12:50:42
 * @LastEditors: Please set LastEditors
 -->
## React-Redux源码分析

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



