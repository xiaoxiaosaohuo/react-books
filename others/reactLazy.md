<!--
 * @Description: In User Settings Edit
 * @Author: your name
 * @Date: 2019-09-05 23:07:52
 * @LastEditTime: 2019-09-08 18:53:34
 * @LastEditors: Please set LastEditors
 -->
## reactLazy

懒加载组件用法示例

fallback 会在OtherComponent组件为完成load之前渲染一个占位的UI，比如loading。

```
const OtherComponent = React.lazy(() => import('./OtherComponent'));

const MyLazy = () => {
  return (
        <React.Suspense fallback={<h1>Still Loading…</h1>}>
            <OtherComponent />
        </React.Suspense>
  );
}

export default MyLazy;
```



```

function lazy(ctor) {
  var lazyType = {
    $$typeof: REACT_LAZY_TYPE,
    _ctor: ctor,
    // React uses these fields to store the result.
    _status: -1,
    _result: null
  };

  return lazyType;
}
```


#### updateSuspenseComponent


待更...