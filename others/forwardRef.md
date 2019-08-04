## Forwarding Refs


##### 组件内使用ref，获取dom元素

```
class App extends Component{
    constructor(){
        super();
        this.myDiv = React.createRef();
    }
    componentDidMount(){
        console.log(this.myDiv.current);
    }
    render(){
        return (
            <div ref={this.myDiv}>ref获取的dom元素</div>
        )
    }
    
}

```

假如想获取组件中子组件中的dom实例，该怎么办呢？答案是```forwardRef```

经典的应用场景是
- 触发深层input的focus（如自动聚焦搜索框）

- 计算元素宽高尺寸（如JS布局方案）

- 重新定位DOM元素（如tooltip）

```

当然之前的方案也可以解决，不过forwardRef API提供了一种更加优雅。

const FancyButton = React.forwardRef((props, ref) => (
    <button ref={ref} className="FancyButton" {...props}>
      {props.children}
    </button>
  ));
  
  // You can now get a ref directly to the DOM button:
class App extends React.Component{
  constructor(props){
    super(props);
    this.ref = React.createRef();
  }
  onClick  = ()=>{
    console.log(this.ref)
  }
  render(){
    return(
      <FancyButton ref={this.ref} onClick={this.onClick}>Click me!</FancyButton>
    )
  }
}

export default App;

```

### createRef

createRef很简单就是返回一个带current属性的对象，
通过current属性可以访问到当前组件的实例。


```
export function createRef(): RefObject {
  const refObject = {
    current: null,
  };
  if (__DEV__) {
    Object.seal(refObject);
  }
  return refObject;
}
```

### forwardRef

参数render是是一个函数，返回一个 React$Node
```
export default function forwardRef<Props, ElementType: React$ElementType>(
  render: (props: Props, ref: React$Ref<ElementType>) => React$Node,
) {
  return {
    $$typeof: REACT_FORWARD_REF_TYPE,
    render,
  };
}
```

### ref的挂载，更新， 卸载

当进入beginWork方法时，如果组件使用了forwardRef,会调用`updateForwardRef`方法处理ref

更新---updateForwardRef
```
function updateForwardRef(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: any,
  nextProps: any,
  renderExpirationTime: ExpirationTime,
) {
  const render = Component.render;
  const ref = workInProgress.ref;

  // 很多逻辑和updateFunctionComponent方法相同，因为render就是一个函数式组件

  let nextChildren;
  prepareToReadContext(workInProgress, renderExpirationTime);
  if (enableFlareAPI) {
    prepareToReadListenerHooks(workInProgress);
  }
  nextChildren = renderWithHooks(
      current,
      workInProgress,
      render,
      nextProps,
      ref,
      renderExpirationTime,
    );

  if (current !== null && !didReceiveUpdate) {
    bailoutHooks(current, workInProgress, renderExpirationTime);
    return bailoutOnAlreadyFinishedWork(
      current,
      workInProgress,
      renderExpirationTime,
    );
  }

  // React DevTools reads this flag.
  workInProgress.effectTag |= PerformedWork;
  reconcileChildren(
    current,
    workInProgress,
    nextChildren,
    renderExpirationTime,
  );
  return workInProgress.child;
}
```

挂载---commitAttachRef

将组件实例赋值ref.current，该方法在commit阶段调用
```
function commitAttachRef(finishedWork: Fiber) {
  const ref = finishedWork.ref;
  if (ref !== null) {
    //组件实例
    const instance = finishedWork.stateNode;
    let instanceToUse;
    switch (finishedWork.tag) {
      //原生组件
      case HostComponent:
        instanceToUse = getPublicInstance(instance);
        break;
      default:
        instanceToUse = instance;
    }
    if (typeof ref === 'function') {
      ref(instanceToUse);
    } else {

      ref.current = instanceToUse;
    }
  }
}

```

卸载---commitDetachRef

将ref的current置为null
```
function commitDetachRef(current: Fiber) {
  const currentRef = current.ref;
  if (currentRef !== null) {
    if (typeof currentRef === 'function') {
      currentRef(null);
    } else {
      currentRef.current = null;
    }
  }
}
```


