## Context API

React 16 提供有四个context相关的API

- createContext
- Provider
- contextType
- Consumer


例子
```

const calculateChangedBits = (currentValue, nextValue) =>
currentValue.name !== nextValue.name ? 0b10 : 0b00;

const UserContext = React.createContext(null,calculateChangedBits);

class App extends React.Component {
  constructor(props) {
    super(props);

    this.state = {
      name: "beforeUpdate"
    };
  }
  setName = () => {
    this.setState(state => ({
      name: state.name === "beforeUpdate" ? "afterUpdate" : "beforeUpdate"
    }));
  };
  render() {
    return (
        <UserContext.Provider value={{ name: this.state.name ,setName:this.setName}}>
         <Indirection>
          <User />
        </Indirection>
        </UserContext.Provider>
    );
  }
}

function User() {
  return (
    <UserContext.Consumer unstable_observedBits={0b10}>
      {({name,setName}) => (
        <button
        onClick={setName}
          >
          {name}
        </button>
      )}
    </UserContext.Consumer>
  );
}
class Indirection extends React.Component {
  shouldComponentUpdate() {
    return false;
  }
  render() {
    return this.props.children;
  }
}


export default App;


```

#### React.createContext()

传入一个初始值/默认值，创建一个 ReactContext,会生成 Provider 与 Consumer，它们是 fiber 中具有特殊类型（$$typeof）的节点，同时是内存共享的，这对后续 Consumer 接收 Provider 的值，以及实现跨层级更新都非常重要。

```
// 创建 React context 赋默认值
const {Consumer,Provider} = React.createContext({
  theme: themes.dark,
  toggleTheme: () => {},
});
```


```
export function createContext<T>(
  defaultValue: T,
  calculateChangedBits: ?(a: T, b: T) => number,
): ReactContext<T> {
  if (calculateChangedBits === undefined) {
    calculateChangedBits = null;
  }

  const context: ReactContext<T> = {
    $$typeof: REACT_CONTEXT_TYPE,
    //用于细粒度的更新控制
    _calculateChangedBits: calculateChangedBits,
     //这里两个currentValue是为了支持多个renderer的
    _currentValue: defaultValue,
    _currentValue2: defaultValue,
    _threadCount: 0,
    // 这里是一个循环引用
    Provider: (null: any),
    Consumer: (null: any),
  };

  context.Provider = {
    $$typeof: REACT_PROVIDER_TYPE,
    _context: context,
  };

  let hasWarnedAboutUsingNestedContextConsumers = false;
  let hasWarnedAboutUsingConsumerProvider = false;
  context.Consumer = context;

  return context;
}
```
Consumer和Provider存在循环引用，为了方便检测这二者是否匹配


### ReactContext.Provider


Provider是context的提供者，我们可以给这个组件传一个value值来覆盖createContext传入的默认值，当value值变化时就会通知到子级的消费者。

当 provider 的 value 变化时，会把当前 provider 的 value 赋值给 ReactContext._currentValue，后面我们的 consum 可以直接从_currentValue去获取最新的值

Provider 所对应的 fiber 节点在创建时，调用updateContextProvider方法
```

function updateContextProvider(
  current: Fiber | null,
  workInProgress: Fiber,
  renderExpirationTime: ExpirationTime,
) {
  const providerType: ReactProviderType<any> = workInProgress.type;
  const context: ReactContext<any> = providerType._context;

  const newProps = workInProgress.pendingProps;
  const oldProps = workInProgress.memoizedProps;

  const newValue = newProps.value;

    //将新的value赋值currentValue
  pushProvider(workInProgress, newValue);

  if (oldProps !== null) {
    const oldValue = oldProps.value;
    //运行传入的calculateChangedBits方法，判断有无改变
    const changedBits = calculateChangedBits(context, newValue, oldValue);
    if (changedBits === 0) {
      // No change. Bailout early if children are the same.
      if (
        oldProps.children === newProps.children &&
        !hasLegacyContextChanged()
      ) {
        return bailoutOnAlreadyFinishedWork(
          current,
          workInProgress,
          renderExpirationTime,
        );
      }
    } else {
        //context 的 value发生了变化就搜索匹配的consumer，找到后安排调度更新
        //修改其expirationTime，并向上遍历，修改祖先的childExpirationTime
      propagateContextChange(
        workInProgress,
        context,
        changedBits,
        renderExpirationTime,
      );
    }
  }

  const newChildren = newProps.children;
  reconcileChildren(current, workInProgress, newChildren, renderExpirationTime);
  return workInProgress.child;
}

export function pushProvider<T>(providerFiber: Fiber, nextValue: T): void {
  const context: ReactContext<T> = providerFiber.type._context;

  if (isPrimaryRenderer) {
    push(valueCursor, context._currentValue, providerFiber);

    context._currentValue = nextValue;
    
  } else {
    push(valueCursor, context._currentValue2, providerFiber);
    context._currentValue2 = nextValue;
  }
}
```

propagateContextChange函数很重要

假如Provider 与 Consumer 之间有一个组件使用了shouldComponentUpdate且返回false,propagateContextChange 函数能保证Consumer能接收到更新后的value。


在propagateContextChange中，以当前fiber为出发点在子树中遍历寻找匹配的Consumer节点，
一旦找到后会不断的回溯，修改祖先节点的childExpirationTime，保证相关的子树能重新render，而不会因为shouldComponentUpdate而失效。

```
export function propagateContextChange(
  workInProgress: Fiber,
  context: ReactContext<mixed>,
  changedBits: number,
  renderExpirationTime: ExpirationTime,
): void {
  let fiber = workInProgress.child;
  if (fiber !== null) {
    // Set the return pointer of the child to the work-in-progress fiber.
    fiber.return = workInProgress;
  }
  while (fiber !== null) {
    let nextFiber;

    // Visit this fiber.
    //在readContext方法中会设置当前fiber的dependencies属性
    const list = fiber.dependencies;
    if (list !== null) {
      nextFiber = fiber.child;

      let dependency = list.firstContext;
      while (dependency !== null) {
        // Check if the context matches.
        //context有变化就调度更新
        if (
          dependency.context === context &&
          (dependency.observedBits & changedBits) !== 0
        ) {
          // Match! Schedule an update on this fiber.

          if (fiber.tag === ClassComponent) {
            // Schedule a force update on the work-in-progress.
            const update = createUpdate(renderExpirationTime, null);
            update.tag = ForceUpdate;
           
            enqueueUpdate(fiber, update);
          }

          if (fiber.expirationTime < renderExpirationTime) {
            fiber.expirationTime = renderExpirationTime;
          }
          let alternate = fiber.alternate;
          if (
            alternate !== null &&
            alternate.expirationTime < renderExpirationTime
          ) {
            alternate.expirationTime = renderExpirationTime;
          }

          scheduleWorkOnParentPath(fiber.return, renderExpirationTime);

          // Mark the expiration time on the list, too.
          if (list.expirationTime < renderExpirationTime) {
            list.expirationTime = renderExpirationTime;
          }

          // Since we already found a match, we can stop traversing the
          // dependency list.
          break;
        }
        dependency = dependency.next;
      }
    } else if (fiber.tag === ContextProvider) {
      // Don't scan deeper if this is a matching provider
      nextFiber = fiber.type === workInProgress.type ? null : fiber.child;
    } else if (
      enableSuspenseServerRenderer &&
      fiber.tag === DehydratedSuspenseComponent
    ) {
      // If a dehydrated suspense component is in this subtree, we don't know
      // if it will have any context consumers in it. The best we can do is
      // mark it as having updates on its children.
      if (fiber.expirationTime < renderExpirationTime) {
        fiber.expirationTime = renderExpirationTime;
      }
      let alternate = fiber.alternate;
      if (
        alternate !== null &&
        alternate.expirationTime < renderExpirationTime
      ) {
        alternate.expirationTime = renderExpirationTime;
      }
      // This is intentionally passing this fiber as the parent
      // because we want to schedule this fiber as having work
      // on its children. We'll use the childExpirationTime on
      // this fiber to indicate that a context has changed.
      scheduleWorkOnParentPath(fiber, renderExpirationTime);
      nextFiber = fiber.sibling;
    } else {
      // Traverse down.
      nextFiber = fiber.child;
    }

    if (nextFiber !== null) {
      // Set the return pointer of the child to the work-in-progress fiber.
      nextFiber.return = fiber;
    } else {
      // No child. Traverse to next sibling.
      nextFiber = fiber;
      while (nextFiber !== null) {
        if (nextFiber === workInProgress) {
          // We're back to the root of this subtree. Exit.
          nextFiber = null;
          break;
        }
        let sibling = nextFiber.sibling;
        if (sibling !== null) {
          // Set the return pointer of the sibling to the work-in-progress fiber.
          sibling.return = nextFiber.return;
          nextFiber = sibling;
          break;
        }
        // No more siblings. Traverse up.
        nextFiber = nextFiber.return;
      }
    }
    fiber = nextFiber;
  }
}
```

### ReactContext.Consumer

ReactContext.Consumer在收到需要更新的时候，会去读取context的currentValue 作为最新的 contextValue，传给子组件，同时Consumer需要建立依赖关系，修改fiber的dependencies属性


```
function updateContextConsumer(
  current: Fiber | null,
  workInProgress: Fiber,
  renderExpirationTime: ExpirationTime,
) {
  let context: ReactContext<any> = workInProgress.type;
  
  const newProps = workInProgress.pendingProps;
  const render = newProps.children;


  prepareToReadContext(workInProgress, renderExpirationTime);
  //读取新的value
  const newValue = readContext(context, newProps.unstable_observedBits);
  let newChildren;
//将新的value传给children
  newChildren = render(newValue);

  reconcileChildren(current, workInProgress, newChildren, renderExpirationTime);
  return workInProgress.child;
}


export function readContext<T>(
  context: ReactContext<T>,
  observedBits: void | number | boolean,
): T {

  if (lastContextWithAllBitsObserved === context) {
    // Nothing to do. We already observe everything in this context.
  } else if (observedBits === false || observedBits === 0) {
    // Do not observe any updates.
  } else {
    let resolvedObservedBits; // Avoid deopting on observable arguments or heterogeneous types.
    if (
      typeof observedBits !== 'number' ||
      observedBits === MAX_SIGNED_31_BIT_INT
    ) {
      // Observe all updates.
      lastContextWithAllBitsObserved = ((context: any): ReactContext<mixed>);
      resolvedObservedBits = MAX_SIGNED_31_BIT_INT;
    } else {
      resolvedObservedBits = observedBits;
    }

    let contextItem = {
      context: ((context: any): ReactContext<mixed>),
      observedBits: resolvedObservedBits,
      next: null,
    };

    if (lastContextDependency === null) {

      // This is the first dependency for this component. Create a new list.
      lastContextDependency = contextItem;
      currentlyRenderingFiber.dependencies = {
        expirationTime: NoWork,
        firstContext: contextItem,
        listeners: null,
        responders: null,
      };
    } else {
      // Append a new context item.
      lastContextDependency = lastContextDependency.next = contextItem;
    }
  }
  return isPrimaryRenderer ? context._currentValue : context._currentValue2;
}
```

### changedBits/observedBits 粒度控制

Context 已经内置让开发者手动控制更新粒度的方式，React.createContext 的第一个参数是 defaultValue，它还拥有第二个参数 calculateChangedBits，它是一个接受 newValue 与 oldValue 的函数，返回值作为 changedBits，在 Provider 中，当 changedBits = 0，将不再触发更新。而在 Consumer 中有一个不稳定的 props，unstable_observedBits，若 Provider 的changedBits & observedBits = 0，也将不触发更新。


- changedBits 是创建context的时候传入的calculateChangedBits返回的结果
- observedBits是consumer 的属性unstable_observedBits，该属性需要是二进制数字




