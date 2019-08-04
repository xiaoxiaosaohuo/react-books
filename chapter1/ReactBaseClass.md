## ReactBaseClasses

在React中常见的基本类就两个

- Component
- PureComponent

这两个基本类的作用就是帮助我们更新组件的state. 比如setState方法等


```
function Component(props, context, updater) {
  this.props = props;
  this.context = context;
  // If a component has string refs, we will assign a different object later.
  this.refs = emptyObject;
  // We initialize the default updater but the real one gets injected by the
  // renderer.
  this.updater = updater || ReactNoopUpdateQueue;
}
Component.prototype.isReactComponent = {};
```

可以看到updater是通过Component构造函数的构造器传入的，
也就是说这个updater会在组件实例化的时候传入，```React.js```这个文件并不负责组件的实例化，具体实例化的工作不同渲染器有不同的实现。

## setState
```
Component.prototype.setState = function(partialState, callback) {
  invariant(
    typeof partialState === 'object' ||
      typeof partialState === 'function' ||
      partialState == null,
    'setState(...): takes an object of state variables to update or a ' +
      'function which returns an object of state variables.',
  );
  this.updater.enqueueSetState(this, partialState, callback, 'setState');
};
```
有两个参数，其中partialState 既可以是一个对象，也可以是一个函数，二者返回的state后续都将和当前state进行合并，callback是setState执行完毕的回调
setState调用时，this.state并不能立即更新，setState不能被保证同步调用，其最终可能进行批处理，所以this.state访问的属性有可能是一个旧值。


后面会专门介绍setState机制


## forceUpdate

```
Component.prototype.forceUpdate = function(callback) {
  this.updater.enqueueForceUpdate(this, callback, 'forceUpdate');
};
```
强制更新，会触发componentWillUpdate和componentDidUpdate`.两个生命周期。一般不建议使用

## PureComponent

```
function ComponentDummy() {}
ComponentDummy.prototype = Component.prototype;

/**
 * Convenience component with default shallow equality check for sCU.
 */
function PureComponent(props, context, updater) {
  this.props = props;
  this.context = context;
  // If a component has string refs, we will assign a different object later.
  this.refs = emptyObject;
  this.updater = updater || ReactNoopUpdateQueue;
}

const pureComponentPrototype = (PureComponent.prototype = new ComponentDummy());
pureComponentPrototype.constructor = PureComponent;
// Avoid an extra prototype jump for these methods.
Object.assign(pureComponentPrototype, Component.prototype);
pureComponentPrototype.isPureReactComponent = true;
```

在实现上PureComponent继承自Component，二者原型上的方法是一样的，只是PureComponent的原型上多了一个属性isPureReactComponent=true。

在判断组件是否应该更新时会调用checkShouldComponentUpdate方法，其中有这样的判断。

```
if (ctor.prototype && ctor.prototype.isPureReactComponent) {
    return !shallowEqual(oldProps, newProps) || !shallowEqual(oldState, newState);
  }
```
ctor就是我们定义组件的构造器，通过判断原型上是否有isPureReactComponent属性来断定是不是PureComponent组件。


在checkShouldComponentUpdate方法中，如果普通Component会调用shouldComponentUpdate，如果是PureComponent则会默认进行一个浅比较，所以PureComponent具有性能优化的作用。

