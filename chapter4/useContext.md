## useContext


useContext 的最大的改变是可以在使用 Consumer 的时候不必在以render props的形式包裹children了。


example

```
//创建一个context parent.js
export const MyContext = React.createContext();

fucntion Parent (){
    return(
        <MyContext.Provider value ='Hello world'>
	    	     <Child />
	    </MyContext.Provider>
    )
}
//child.js
import {MyContext} from './parent'
function Child(){
    const value = useContext(MyContext);
	return (
	    <p>{value}</p>
	)
}

```

useContext和readContext是同一个方法，并不区分mount和update阶段,可以看context的介绍
```
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