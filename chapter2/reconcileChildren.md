## reconcileChildren

reconcileChildren方法最终调用的是reconcileChildFibers，mountChildFibers和reconcileChildFibers方法是一样的，唯一的区别是生成这个方法的时候的一个参数不同

```
var reconcileChildFibers = ChildReconciler(true);
var mountChildFibers = ChildReconciler(false);
```

这个参数叫shouldTrackSideEffects，他的作用是判断是否要增加一些effectTag，主要是用来优化初次渲染的;


