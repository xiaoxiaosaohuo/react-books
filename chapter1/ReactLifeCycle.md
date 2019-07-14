## React生命周期和反模式


![结构](../images/lifecycle.png)

在React 16.4之后 引入fiber架构有些生命周期会慢慢淘汰掉

#### 预废弃

- componentWillMount
- componentWillUpdate
- componentWillReceiveProps

#### 已新增
- getDerivedStateFromProps
- getSnapshotBeforeUpdate
- UNSAFE_componentWillMount 
- UNSAFE_componentWillReceiveProps 
- UNSAFE_componentWillUpdate
  

目前新增了三个生命周期，其实也就是在预废弃的周期函数前加一个USAFE_前缀。



在React 16.4之前，React的渲染机制遵循同步渲染: 
-  首次渲染: willMount > render > didMount， 
-  props更新时: receiveProps > shouldUpdate > willUpdate > render > didUpdate 
-  state更新时: shouldUpdate > willUpdate > render > didUpdate 
- 卸载时: willUnmount
从React 17 开始，渲染机制将会发生颠覆性改变，这个新方式就是 **Async Render**。


首先，async render不是那种服务端渲染，比如发异步请求到后台返回newState甚至新的html，这里的async render还是限制在React作为一个View框架的View层本身。

预废弃的三个生命周期函数都发生在虚拟dom的构建期间，也就是真正的commit更新且进行绘制之前，在将来的React 17中，在dom真正render之前，React中的调度机制可能会不定期的去查看有没有更高优先级的任务，如果有，就打断当前的周期执行函数(哪怕已经执行了一半)，等高优先级任务完成，再回来重新执行之前被打断的周期函数。这种新机制对现存周期函数的影响就是它们的调用时机变的复杂而不可预测，这也就是为什么”UNSAFE”。


## static getDerivedStateFromProps(props, state)

它是作为componentWillReceiveProps的代替品出现，而在React 16 之前componentWillReceiveProps是一个最容易被滥用(misuse)的周期函数。

这个生命周期一般被用来更新组件的状态，或者在组件的属性发生变化时做一些处理逻辑,从而避免掉一次额外的render。

```
static getDerivedStateFromProps(props, state)
```

getDerivedStateFromProps在调用render方法之前调用，包括初始mount和后续更新。它应返回一个对象来更新状态，或者返回null以不更新任何内容.

####  调用时机差异
getDerivedStateFromProps更全能，无论是mounting还是updating都会被触发。componentWillReceiveProps只会updating阶段，并且是父组件触发的render才被调用。

#### derived state

由于组件的props改变而引发了state改变，这个state就是derived state. derived from props.  

####  static

这个方法是静态的，不能访问到组件的实例，无法在方法中使用this。

## 什么时候使用Derived State

```getDerivedStateFromProps``` 这个静态方法使组件能够根据props的更改来更新其内部状态。但是经常会不恰当的使用该方法，会导致组件频繁的更新。

总结起来有两点

- 无条件的根据props来更新state
- 在props和state不匹配的时候更新state


![结构](../images/drivedstate.png)



## 例子

下面的组件接受一个prop---一系列的items，根据用户输入的查询条件渲染items，我们使用derived state来存储过滤出的list.

```
class Example extends Component {
  state = {
    filterText: "",
  };

  // *******************************************************
  // NOTE: this example is NOT the recommended approach.
  // See the examples below for our recommendations instead.
  // *******************************************************

  static getDerivedStateFromProps(props, state) {
    // Re-run the filter whenever the list array or filter text change.
    // Note we need to store prevPropsList and prevFilterText to detect changes.
    if (
      props.list !== state.prevPropsList ||
      state.prevFilterText !== state.filterText
    ) {
      return {
        prevPropsList: props.list,
        prevFilterText: state.filterText,
        filteredList: props.list.filter(item => item.text.includes(state.filterText))
      };
    }
    return null;
  }

  handleChange = event => {
    this.setState({ filterText: event.target.value });
  };

  render() {
    return (
      <Fragment>
        <input onChange={this.handleChange} value={this.state.filterText} />
        <ul>{this.state.filteredList.map(item => <li key={item.id}>{item.text}</li>)}</ul>
      </Fragment>
    );
  }
}
```

这种实现方式避免了不必要的重复计算，但是比较复杂，必须分开跟踪props和state变化来保证得到正确的filteredList.

在这个例子中，我们可以使用PureComponent，将过滤逻辑放在render方法中

```
class Example extends PureComponent {
  // State only needs to hold the current filter text value:
  state = {
    filterText: ""
  };

  handleChange = event => {
    this.setState({ filterText: event.target.value });
  };

  render() {
    // The render method on this PureComponent is called only if
    // props.list or state.filterText has changed.
    const filteredList = this.props.list.filter(
      item => item.text.includes(this.state.filterText)
    )

    return (
      <Fragment>
        <input onChange={this.handleChange} value={this.state.filterText} />
        <ul>{filteredList.map(item => <li key={item.id}>{item.text}</li>)}</ul>
      </Fragment>
    );
  }
}
```

看起来清晰简单了很多，但是也不够好，如果列表项很大的话可能会比较耗时，而且如果其他props发生变化，PureComponent可能不会阻止rerender。

我们可以使用memoization

```
import memoize from "memoize-one";

class Example extends Component {
  // State only needs to hold the current filter text value:
  state = { filterText: "" };

  // Re-run the filter whenever the list array or filter text changes:
  filter = memoize(
    (list, filterText) => list.filter(item => item.text.includes(filterText))
  );

  handleChange = event => {
    this.setState({ filterText: event.target.value });
  };

  render() {
    // Calculate the latest filtered list. If these arguments haven't changed
    // since the last render, `memoize-one` will reuse the last return value.
    const filteredList = this.filter(this.props.list, this.state.filterText);

    return (
      <Fragment>
        <input onChange={this.handleChange} value={this.state.filterText} />
        <ul>{filteredList.map(item => <li key={item.id}>{item.text}</li>)}</ul>
      </Fragment>
    );
  }
}
```



## 使用Derived State是常见的错误

名词“受控”和“非受控”通常用来指代表单的 inputs，但是也可以用来描述数据频繁更新的组件。用 props 传入数据的话，组件可以被认为是受控（因为组件被父级传入的 props 控制）。数据只保存在组件内部的 state 的话，是非受控组件（因为外部没办法直接控制 state）。


1. 直接复制 prop 到 state

最常见的误解就是 getDerivedStateFromProps 和 componentWillReceiveProps 只会在 props “改变”时才会调用。实际上只要父级重新渲染时，这两个生命周期函数就会重新调用，不管 props 有没有“变化”。所以，在这两个方法内直接复制（unconditionally）props 到 state 是不安全的。这样做会导致状态更新丢失。

这个 EmailInput 组件复制 props 到 state：

```
class EmailInput extends Component {
  state = { email: this.props.email };

  render() {
    return <input onChange={this.handleChange} value={this.state.email} />;
  }

  handleChange = event => {
    this.setState({ email: event.target.value });
  };

  componentWillReceiveProps(nextProps) {
    // 这会覆盖所有组件内的 state 更新！
    // 不要这样做。
    this.setState({ email: nextProps.email });
  }
}
```

state 的初始值是 props 传来的，当在 <input> 里输入时，修改 state。但是如果父组件重新渲染，我们输入的所有东西都会丢失！

使用 shouldComponentUpdate ，比较 props 的 email 是不是修改再决定要不要重新渲染。但是在实践中，一个组件会接收多个 prop，任何一个 prop 的改变都会导致重新渲染和不正确的状态重置。加上行内函数和对象 prop，创建一个完全可靠的 shouldComponentUpdate 会变得越来越难。而且 shouldComponentUpdate 的最佳实践是用于性能提升，而不是改正不合适的派生 state。

2. 在 props 变化后修改 state

```
componentWillReceiveProps(nextProps) {
    // 只要 props.email 改变，就改变 state
    if (nextProps.email !== this.props.email) {
      this.setState({
        email: nextProps.email
      });
    }
  }
```

但是仍然有个问题。想象一下，如果这是一个密码输入组件，拥有同样 email 的两个账户进行切换时，这个输入框不会重置（用来让用户重新登录）。因为父组件传来的 prop 值没有变化！这会让用户非常惊讶，因为这看起来像是帮助一个用户分享了另外一个用户的密码，([查看这个示例](https://codesandbox.io/s/mz2lnkjkrx))。



> **这两者的关键在于，任何数据，都要保证只有一个数据来源，而且避免直接复制它**。

## 建议的模式

1. 完全可控的组件

从组件里删除 state。如果 prop 里包含了 email，我们就没必要担心它和 state 冲突。我们甚至可以把 EmailInput 转换成一个轻量的函数组件：

```
function EmailInput(props) {
  return <input onChange={props.onChange} value={props.email} />;
}
```
但是如果我们仍然想要保存临时的值，则需要父组件手动执行保存这个动作

2. 有 key 的非可控组件

我们可以使用 key 这个特殊的 React 属性。当 key 变化时， React 会创建一个新的而不是更新一个既有的组件。 Keys 一般用来渲染动态列表，但是这里也可以使用。在这个示例里，当用户输入时，我们使用 user ID 当作 key 重新创建一个新的 email input 组件：

```
<EmailInput
  defaultEmail={this.props.user.email}
  key={this.props.user.id}
/>
```

大部分情况下，这是处理重置 state 的最好的办法。

> 这听起来很慢，但是这点的性能是可以忽略的。如果在组件树的更新上有很重的逻辑，这样反而会更快，因为省略了子组件 diff。