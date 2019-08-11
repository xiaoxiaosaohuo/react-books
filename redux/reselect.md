<!--
 * @Description: In User Settings Edit
 * @Author: 金鑫
 * @Date: 2019-07-29 14:27:48
 * @LastEditTime: 2019-08-10 16:08:23
 * @LastEditors: Please set LastEditors
 -->
## Reselect

在了解Reselect 之前需要了解一下记忆函数

所谓记忆函数，即函数能够记住最后一次调用的值和结果，如果下一次调用参数无变化，就返回上一次的结果。

```
function memoize(f) {
    var cache = {};
    return function(){
        var key = arguments.length + Array.prototype.join.call(arguments, ",");
        if (key in cache) {
            return cache[key]
        }
        else return cache[key] = f.apply(this, arguments)
    }
}

```
Reselect 最核心的作用就是缓存记忆。

### Reselect的功能

- Selectors可以计算派生数据，允许Redux存储最小可能状态。
- Selectors不会重新计算除非参数发生了变化
- Selectors是可以组合的，可以作为其他Selectors的输入


### Reselect 基本用法

```
import { connect } from 'react-redux';
import { createSelector } from 'reselect';
import { toggleTodo } from '../actions';
import TodoList from '../components/TodoList';


const getVisibilityFilter = (state) => state.visibilityFilter
const getTodos = (state) => state.todos

export const getVisibleTodos = createSelector(
  [ getVisibilityFilter, getTodos ],
  (visibilityFilter, todos) => {
    switch (visibilityFilter) {
      case 'SHOW_ALL':
        return todos
      case 'SHOW_COMPLETED':
        return todos.filter(t => t.completed)
      case 'SHOW_ACTIVE':
        return todos.filter(t => !t.completed)
    }
  }
)
const mapStateToProps = (state) => {
  return {
    todos: getVisibleTodos(state)
  }
}
```

##### 默认相等检查函数

采用的是全等比较
```
function defaultEqualityCheck(a, b) {
  return a === b
}
```

##### 参数比较
- equalityCheck 比较函数
- prev 前一个值
- next 下一个值
```
function areArgumentsShallowlyEqual(equalityCheck, prev, next) {
  if (prev === null || next === null || prev.length !== next.length) {
    return false
  }

  // Do this in a for loop (and not a `forEach` or an `every`) so we can determine equality as fast as possible.
  // 使用for循环而不是every或forEach，可以尽快地进行相等判断，发现不等可以提前结束循环
  const length = prev.length
  for (let i = 0; i < length; i++) {
    if (!equalityCheck(prev[i], next[i])) {
      return false
    }
  }

  return true
}
```

##### defaultMemoize

```

export function defaultMemoize(func, equalityCheck = defaultEqualityCheck) {
  let lastArgs = null;// 上一次调用的参数
  let lastResult = null;// 上一次调用的结果
  // we reference arguments instead of spreading them for performance reasons
  return function () {
    //比较两次调用参数是否相同
    if (!areArgumentsShallowlyEqual(equalityCheck, lastArgs, arguments)) {
      // apply arguments instead of spreading for performance.
      lastResult = func.apply(null, arguments)
    }
    lastArgs = arguments
    return lastResult
  }
}

````

##### 依赖函数校验器

检查参数确保每个元素均是函数
第一个参数可以是一个数组
```

function getDependencies(funcs) {
  const dependencies = Array.isArray(funcs[0]) ? funcs[0] : funcs

  if (!dependencies.every(dep => typeof dep === 'function')) {
    const dependencyTypes = dependencies.map(
      dep => typeof dep
    ).join(', ')
    throw new Error(
      'Selector creators expect all input-selectors to be functions, ' +
      `instead received the following types: [${dependencyTypes}]`
    )
  }

  return dependencies
}

```
##### createSelectorCreator 选择器函数生成器


createSelectorCreator 接受一个默认的缓存函数，返回一个函数用于接受选择器

所谓的选择器就是接受state作为参数的函数，其返回值是mapStateToProps需要的结果

Reselect把mapStateToProps的工作分为两部分
- 从输入参数state抽取第一层结果，将这个结果和之前的比较，如果相同就直接返回之前的结果
- 如果不同就重新计算

```

export function createSelectorCreator(memoize, ...memoizeOptions) {
  return (...funcs) => {
    let recomputations = 0; //统计是计算次数

    const resultFunc = funcs.pop(); //结果函数
    const dependencies = getDependencies(funcs) //依赖函数
    //将resultFunc 包装成记忆函数
    const memoizedResultFunc = memoize(
      function () {
        recomputations++
        // apply arguments instead of spreading for performance.
        return resultFunc.apply(null, arguments)
      },
      ...memoizeOptions
    )

    // If a selector is called with the exact same arguments we don't need to traverse our dependencies again.

    const selector = memoize(function () {
      const params = []// 要记忆的映射函数计算结果
      const length = dependencies.length

      for (let i = 0; i < length; i++) {
        // apply arguments instead of spreading and mutate a local list of params for performance.
        params.push(dependencies[i].apply(null, arguments))
      }

      // apply arguments instead of spreading for performance.
      // 把映射函数的计算结果传给步骤二的结果计算函数
      return memoizedResultFunc.apply(null, params)
    })

    selector.resultFunc = resultFunc
    selector.dependencies = dependencies
    selector.recomputations = () => recomputations
    selector.resetRecomputations = () => recomputations = 0
    return selector
  }
}

export const createSelector = createSelectorCreator(defaultMemoize)


```
##### createStructuredSelector

createStructuredSelector 的inputSelectors是一个对象，返回一个结构化的selector。该对象具有与inputSelectors参数相同的键，但是其结果是选择器执行的结果


```
export function createStructuredSelector(selectors, selectorCreator = createSelector) {
  if (typeof selectors !== 'object') {
    throw new Error(
      'createStructuredSelector expects first argument to be an object ' +
      `where each property is a selector, instead received a ${typeof selectors}`
    )
  }
  const objectKeys = Object.keys(selectors)
  return selectorCreator(
    objectKeys.map(key => selectors[key]),
    (...values) => {
      return values.reduce((composition, value, index) => {
        composition[objectKeys[index]] = value
        return composition
      }, {})
    }
  )
}
```
example:
```
const mySelectorA = state => state.a
const mySelectorB = state => state.b

const structuredSelector = createStructuredSelector({
  x: mySelectorA,
  y: mySelectorB
})

const result = structuredSelector({ a: 1, b: 2 }) // will produce { x: 1, y: 2 }
```
