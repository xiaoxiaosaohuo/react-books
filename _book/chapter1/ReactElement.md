我们平时写React 都是用的JSX

JSX为创建元素提供了一种语法糖，babel会帮我们调用 React.createElement对JSX语法进行转换。


```
render(){
  return(
    <div>
    <input value="foo" type="text/>
    <span onClick="(e)=>alert("Hello)">click me</span>
    </div>
  )
}
--------调用createElemet方法--------
 createElement(
    "div",
    {id:"container"},
    createElement(
        "input",
        {value:"foo",type:"text"}
        ),
    createElement(
        "span",
        {onClick:e=> alert("Hello React")},
        children:"click me"
    ) 
)
--------creteElement内部调用ReactElement方法,最终处理成普通的对象。

{
    type:"div",
    props:{
        id:"container",
        children:[
            {
                type:"input",
                props:{value:"foo",type:"text"}
            },
            {
                type:"span"
                props:{
                    onClick:e=> alert("Hello React"),
                    children:"click me"
                }
            }
        ]
    }
}
```

上面只有type,props这几个属性，其实还有很多属性。


```
var ReactElement = function (type, key, ref, self, source, owner, props) {
  var element = {
    // This tag allows us to uniquely identify this as a React Element
    $$typeof: REACT_ELEMENT_TYPE,

    // Built-in properties that belong on the element
    type: type,
    key: key,
    ref: ref,
    props: props,

    // Record the component responsible for creating this element.
    _owner: owner
  };
```

- type类型，用于判断如何创建节点
- props新的属性内容
- key 唯一标识
- ref refs 提供了一种方式，允许我们访问 DOM 节点或在 render 方法中创建的 React 元素。
- $$typeof用于确定是否属于ReactElement

其实这就是virtual Dom ，这种类型的数据和原生dom是一一对应的，React在reconcilation过程中会对比Virtual dom 的信息，找出差异，然后再运用到真是的dom上，比如新建，更新，删除等。


### 指定 React 元素类型
JSX 标签的第一部分指定了 React 元素的类型。

大写字母开头的 JSX 标签意味着它们是 React 组件。这些标签会被编译为对命名变量的直接引用，所以，当你使用 JSX <Foo /> 表达式时，Foo 必须包含在作用域内。

### React 必须在作用域内

由于 JSX 会编译为 React.createElement 调用形式，所以 React 库也必须包含在 JSX 代码作用域内。

例如，在如下代码中，虽然 React 和 CustomButton 并没有被直接使用，但还是需要导入：
```
import React from 'react';
import CustomButton from './CustomButton';

function WarningButton() {
  // return React.createElement(CustomButton, {color: 'red'}, null);
  return <CustomButton color="red" />;
}
```
如果你不使用 JavaScript 打包工具而是直接通过 <script> 标签加载 React，则必须将 React 挂载到全局变量中。


