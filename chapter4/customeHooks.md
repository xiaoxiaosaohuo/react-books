## 自定义hooks

自定义Hook其实也是一个JavaScript函数，其名称以use开头，可以调用其他Hook。自定义挂钩是一种自然遵循Hooks设计的约定，最好以use开头。

useMyucer.js
```
import React, { useState } from 'react';

function useMyucer(reducer, initialState) {
    const [state, setState] = useState(initialState);

    function dispatch(action) {
        const nextState = reducer(state, action);
        setState(nextState);
    }

    return [state, dispatch];
}

```
使用自定义的reducer
```
import React from 'react';
import useMyucer from './useMyucer';

const initialState = {count: 0};

function reducer(state, action) {
    switch (action.type) {
        case 'increment':
            return {count: state.count + 1};
        case 'decrement':
            return {count: state.count - 1};
        default:
            throw new Error();
    }
}

export default () => {
    const [state, dispatch] = useMyucer(reducer, initialState);
    return (
        <>
            Count: {state.count}
            <button onClick={() => dispatch({type: 'increment'})}>+</button>
            <button onClick={() => dispatch({type: 'decrement'})}>-</button>
        </>
    );
}

```

hooks非常灵活，可以编写自定义Hook，涵盖广泛的用例，如表单处理，动画，声明订阅，计时器等等 ，极大的方便将组件逻辑提取到可重用的函数中。