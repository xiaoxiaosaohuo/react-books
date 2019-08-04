## workInProgress 和alternate 双缓冲池技术
在第一次render之后，React生成了一个fiber tree代表着当前应用的状态，这棵树就是current，当开始更新的时候，会创建一个workInProgress tree，代表未来会应用到屏幕上的状态。

所有的工作都在workInProgress tree上进行，React遍历current tree,对每一个存在的fiber node 会创建一个workInProgress tree，其数据来自render方法返回的react element。一旦所有的工作完成了且workInProgress tree被render到屏幕上，workInProgress tree就是current tree了

```
 function createWorkInProgress(current, pendingProps, expirationTime) {
  var workInProgress = current.alternate;
  if (workInProgress === null) {

   workInProgress = createFiber(current.tag, pendingProps, current.key, current.mode);
    workInProgress.elementType = current.elementType;
    workInProgress.type = current.type;
    workInProgress.stateNode = current.stateNode;
   workInProgress.alternate = current;
    current.alternate = workInProgress;
}else{
//更新属性
 workInProgress.pendingProps = pendingProps;
workInProgress.child = current.child;
  workInProgress.memoizedProps = current.memoizedProps;
  workInProgress.memoizedState = current.memoizedState;
  workInProgress.updateQueue = current.updateQueue;
workInProgress.sibling = current.sibling;
  workInProgress.index = current.index;
  workInProgress.ref = current.ref;
}
return workInProgress;
```
当首次渲染时，只有一个root节点，root.current就是当前的rootFirber node(我们将其标记为root1) ,会从root节点开始从无到有创建一颗workInProgress树。

workInProgress树中只有root节点的alternate是存在的，其他节点由于一开始就不存在（root本来已经渲染到屏幕上），所以会创建一个对应的fiber node 并且赋值给workInProgress.child,最终形成一颗tree，除了root.current存在alternate，其他节点都不存在。

这里原因是在调和children时，首次渲染current不存在，就会调用对应的方法创建的fiber；


![workInProgress1](https://user-images.githubusercontent.com/15315816/50039932-0b770080-0076-11e9-8b37-ad520b8eef29.png)


当渲染完毕，将root.current指向workInProgress。此时root为root2
```
root.current=workInProgress

```
再次渲染时，root.current相当于上一次的workInProgress，所以root.current.alternate指向root1；
其他的节点会在current树的基础上创建一个workInProgress树

```
workInProgress.alternate = current;
 current.alternate = workInProgress;
```

当再次渲染完毕 将root.current指向workInProgress。此时root为root3；

很明显 root3.current.alternate 指向的是root2.

一个节点能持有当前渲染的状态和上一次渲染的状态，即最多同时存在一棵树的两个版本，当前版本和上一个版本，而且alertnate指向的node是懒创建的，在以后的更新中，如果节点只是更新属性的话，会重用fiber 对象而不会再次创建，有利于节省空间。


![workInProgress1](https://user-images.githubusercontent.com/15315816/50039688-15e2cb80-0071-11e9-8661-5c0ed0878d50.png)