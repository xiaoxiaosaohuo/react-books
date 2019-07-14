## ReactRoot

```
/ ReactRoot构造函数

function ReactRoot(container: DOMContainer, hydrate: boolean) {
    // createContainer是react-reconciler中的方法,从container创建FiberRoot对象
  const root = createContainer(container, ConcurrentRoot, hydrate);
  // 将FiberRoot对象挂载this的_internalRoot属性上
  this._internalRoot = root;
}

ReactRoot.prototype.render = ReactSyncRoot.prototype.render = function(
  children: ReactNodeList,
  callback: ?() => mixed,
): Work {
  const root = this._internalRoot;
  const work = new ReactWork();
  callback = callback === undefined ? null : callback;
  if (__DEV__) {
    warnOnInvalidCallback(callback, 'render');
  }
  if (callback !== null) {
    work.then(callback);
  }
  updateContainer(children, root, null, work._onCommit);
  return work;
};

ReactRoot.prototype.unmount = ReactSyncRoot.prototype.unmount = function(
  callback: ?() => mixed,
): Work {
  const root = this._internalRoot;
  const work = new ReactWork();
  callback = callback === undefined ? null : callback;
  if (__DEV__) {
    warnOnInvalidCallback(callback, 'render');
  }
  if (callback !== null) {
    work.then(callback);
  }
  updateContainer(null, root, null, work._onCommit);
  return work;
};

// Sync roots cannot create batches. Only concurrent ones.
ReactRoot.prototype.createBatch = function(): Batch {
  const batch = new ReactBatch(this);
  const expirationTime = batch._expirationTime;

  const internalRoot = this._internalRoot;
  const firstBatch = internalRoot.firstBatch;
  if (firstBatch === null) {
    internalRoot.firstBatch = batch;
    batch._next = null;
  } else {
    // Insert sorted by expiration time then insertion order
    let insertAfter = null;
    let insertBefore = firstBatch;
    while (
      insertBefore !== null &&
      insertBefore._expirationTime >= expirationTime
    ) {
      insertAfter = insertBefore;
      insertBefore = insertBefore._next;
    }
    batch._next = insertBefore;
    if (insertAfter !== null) {
      insertAfter._next = batch;
    }
  }

  return batch;
};
```

## createContainer
从ReactRoot中， 我们把createContainer返回值赋给了 实例的_internalRoot。

createContainer实际上是直接返回了createFiberRoot, 而createFiberRoot则是通过createHostRootFiber函数的返回值uninitializedFiber，并将其赋值在root对象的current上， 这里需要注意一个点就是，uninitializedFiber的stateNode的值是root， 即他们互相引用

```
function createContainer(containerInfo, isAsync, hydrate) {
  return createFiberRoot(containerInfo, isAsync, hydrate);
}
function createFiberRoot(containerInfo, isAsync, hydrate) {
  // 创建hostRoot并赋值给uninitiallizedFiber
  var uninitializedFiber = createHostRootFiber(isAsync);
  // 互相引用
  var root = void 0;
  root = {
      current: uninitializedFiber,
      ...
  };
 uninitializedFiber.stateNode = root; 
```

在这里， 整理一下各个实例的关系， 

 - root为ReactRoot实例，

- root._internalRoot 即为fiberRoot实例，

- root._internalRoot.current即为Fiber实例，

- root._internalRoot.current.stateNode = root._internalRoot

### ReactRoot.prototype.render 

render方法是在legacyRenderSubtreeIntoContainer调用的

它主要做了以下几件事

- new ReactWork -> work
- 调用work的then方法
- 调用updateContainer
- 返回work


## ReactWork

ReactWork的方法在第一次render时都有可能被调用到，如下代码为ReactWork类的定义

```
function ReactWork() {
  this._callbacks = null;
  this._didCommit = false;
  // TODO: Avoid need to bind by replacing callbacks in the update queue with
  // list of Work objects.
  this._onCommit = this._onCommit.bind(this);
}
ReactWork.prototype.then = function(onCommit: () => mixed): void {
    // 第一次render调用then时为false, 不走这里
  if (this._didCommit) { 
    onCommit();
    return;
  }
  let callbacks = this._callbacks;
  // 第一次render是调用走这里
  if (callbacks === null) {
    callbacks = this._callbacks = [];
  }
  callbacks.push(onCommit);
};
ReactWork.prototype._onCommit = function(): void {
    // 第一次render不走这里
  if (this._didCommit) {
    return;
  }
  this._didCommit = true;
  // 这个callbacks是调用.then方法是传进去的函数
  const callbacks = this._callbacks;
  if (callbacks === null) {
    return;
  }
  // TODO: Error handling.
  //将所有的callback调用一下
  for (let i = 0; i < callbacks.length; i++) {
    const callback = callbacks[i];
    invariant(
      typeof callback === 'function',
      'Invalid argument passed as callback. Expected a function. Instead ' +
        'received: %s',
      callback,
    );
    callback();
  }
};

```


- then方法

then方法调用时是```work.then(callback)```,callback是ReactDOM的第三个参数
then方法的作用就是维护一个_callbacks队列，每次都将传进去的函数入队
- _onCommit方法

这个方法的调用代码是```updateContainer(children, root, null, work._onCommit)```,其实是updateContainer的最后一个参数。
在这个里边将_didCommit置为true，将_callbacks中的方法都执行一遍。


接下来的重点是分析updateContainer这个方法，ReactWork的then方法是将callback入队，_onCommit是执行_callbacks中的所有方法，而调用_onCommit的是在updateContainer中，updateContainer实在ReactRoot.render方法中调用的，因此updateContainer应该是一个非常重要的东西。另外，ReactRoo.render方法是在unbatchedUpdates的回调函数中调用的，unbatchedUpdates也是一个参与后面调度的关键。