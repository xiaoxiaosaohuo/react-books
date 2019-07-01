在react.js文件中对外暴露出了React的一些API

```
var React = {
  Children: {
    map: mapChildren,
    forEach: forEachChildren,
    count: countChildren,
    toArray: toArray,
    only: onlyChild
  },

  createRef: createRef,
  Component: Component,
  PureComponent: PureComponent,

  createContext: createContext,
  forwardRef: forwardRef,
  lazy: lazy,
  memo: memo,

  useCallback: useCallback,
  useContext: useContext,
  useEffect: useEffect,
  useImperativeHandle: useImperativeHandle,
  useDebugValue: useDebugValue,
  useLayoutEffect: useLayoutEffect,
  useMemo: useMemo,
  useReducer: useReducer,
  useRef: useRef,
  useState: useState,

  Fragment: REACT_FRAGMENT_TYPE,
  StrictMode: REACT_STRICT_MODE_TYPE,
  Suspense: REACT_SUSPENSE_TYPE,

  createElement: createElementWithValidation,
  cloneElement: cloneElementWithValidation,
  createFactory: createFactoryWithValidation,
  isValidElement: isValidElement,

  version: ReactVersion,

  unstable_ConcurrentMode: REACT_CONCURRENT_MODE_TYPE,
  unstable_Profiler: REACT_PROFILER_TYPE,

  __SECRET_INTERNALS_DO_NOT_USE_OR_YOU_WILL_BE_FIRED: ReactSharedInternals
};
```
### Children

这个对象提供一些处理props.children的方法，如常见的
```
this.props.children.map
this.props.children.forEach
this.props.only
```

### createRef

新的ref用法，React即将抛弃<div ref="myDiv" />这种string ref的用法，将来你只能使用两种方式来使用ref


```
class App extends React.Component{

  constructor() {
    this.ref = React.createRef()
  }

  render() {
    return <div ref={this.ref} />
    // or
    return <div ref={(node) => this.divRef = node} />
  }

}
```

### Component & PureComponent

这两个组件基本相同，只是PureComponent组件的原型上多了一个标识

```
pureComponentPrototype.isPureReactComponent = true;

```
如果是PureComponent的话就shallowEqual比较state和props。

