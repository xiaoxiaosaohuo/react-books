## useState

useState这个API挂载在React上的，在React.js中可看到useState实现如下，
dispatcher通过resolveDispatcher()来获取，这个函数同样也很简单，只是将ReactCurrentDispatcher.current的值赋给了dispatcher。ReactCurrentDispatcher在前面提到过，会根据mount阶段还是update阶段有不同的dispatcher实现。

```
function useState(initialState) {
  var dispatcher = resolveDispatcher();
  return dispatcher.useState(initialState);
}

function resolveDispatcher() {
  var dispatcher = ReactCurrentDispatcher.current;
  
  return dispatcher;
}
```

useState(xxx) 等价于 ReactCurrentDispatcher.current.useState(xxx)