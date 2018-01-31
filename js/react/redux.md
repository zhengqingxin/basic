# Redux 相关概念及源码分析

## 划重点：
* 三个概念：(state, action) => state
  * store
  * action，例如：`{type: 'COMPLETE_TODO',index: 1}`
  * reducer，只是一些纯函数，它接收先前的 state 和 action，并返回新的 state。
* 三个原则：单一数据源，State 是只读的，使用纯函数执行修改

## 概念相关源码

### createStore
首先我们先看下 createStore 的参数列表及其返回值：
```js
export default function createStore(reducer, preloadedState, enhancer) {
  // 第一个参数reducer（function类型），preloadedState是初始state，enhancer参见下面解释
  // 第二，第三参数可为空，可互换，这里不列源码了

  // ...

  // 返回一个包含一大堆函数的对象，具体作用我们后面看，这里我们首先知道它是返回了一个对象
  return {
    dispatch,
    subscribe,
    getState,
    replaceReducer,
    [$$observable]: observable
  }
}
```
### Store Enhancers
enhancers 英语译为“增强剂”，其实作用也的确如此。用一个表达式来解释：
```js
// f方法对 g 方法做了一些功能增强
h(x) = f(g(x));
```
例如：
```js
const g = (t)=>{
  console.log(`do ${t} task`);
}
const f = (func) => {
  console.log('add enhancers');
  return func;
}
const h = f(g);
h('first');
```
其实 Store Enhancers 也就是这个概念，来看相关源码：
```js
export default function createStore(reducer, preloadedState, enhancer) {
  // ...
  if (typeof enhancer !== 'undefined') {
    if (typeof enhancer !== 'function') {
      throw new Error('Expected the enhancer to be a function.')
    }
    return enhancer(createStore)(reducer, preloadedState)
  }
  // ...
}
```
从源码中可以看出两点：
1. redux 会把 `createStore` 函数传给我们写的 enhancer
2. 同时 `reducer`,`preloadedState`参数也会传给我们

不知道这样是不是很清晰了？我们来写一个例子，功能类似简化版的 [redux-logger]('https://github.com/evgenyrodionov/redux-logger')，可以在每个`dispatch`时打印日志。
```js
function createLogger(createStore){
  return (reducers,initialState) => {
    const store = createStore(reducers, initialState);
    function dispatch(action){
      console.log(`dispatch an action: ${JSON.stringify(action)}`);
      const res = store.dispatch(action);
      const newState = store.getState();
      console.log(`current state: ${JSON.stringify(newState)}`);
      return res;
    }
    return {...store,dispatch};
  }
}
var store = Redux.createStore(counter,createLogger);
```

这样就实现了一个记录日志的功能，每次 `dispatch` 的时候都会记录。
有些人可能想到了，enhancer权力好像有些大？是的，它可以增强 store 的行为，同时也可能破坏它，所以大家写 enhancers 的时候一定要注意：__不要破坏了Redux的原有工作流__。比如上面例子中这行代码就保证了 Redux 原本的工作流。
```js
const res = store.dispatch(action);
```

### Middleware
可能会有人还不知道 Middleware 的含义，还是来解释一下吧，说洋葱模型可能太文艺了，举这么个例子吧，超级玛丽为了救公主要经历 1-1,1-2,1-3,1-4...8-4 关，才能救到公主，这里的每一关就可以理解成一个Middleware，只不过现在我们是游戏制作人，关卡由我们来设定。

如果你看了上面的实例，可能会想 [redux-logger]('https://github.com/evgenyrodionov/redux-logger') 不是一个 Middleware 吗？和 store enhancers 是什么关系啊？我们先来看一下在 redux 中如何定义一个 Middleware，举官网上的例子：
```js
import { createStore, combineReducers, applyMiddleware } from 'redux'
let todoApp = combineReducers(reducers)
let store = createStore(
  todoApp,
  applyMiddleware(logger, crashReporter)
)
```
大家再回想上面例子的代码：
```js
var store = Redux.createStore(counter,createLogger);
```
是不是很相似呢？没错，原理都是一个。这里调用了 Redux 提供的 applyMiddleware 方法，从而实现了我们上面的过程。所以，我把这里的 Middleware 归纳成 __一个只可以扩展 dispatch 方法的 Store Enhancers__。下面列出 applyMiddleware 方法的源码吧，很简单相信大家都可以看懂。
```js
export default function applyMiddleware(...middlewares) {
  return createStore => (...args) => {
    const store = createStore(...args)
    let dispatch = () => {
      throw new Error(
        `Dispatching while constructing your middleware is not allowed. ` +
          `Other middleware would not be applied to this dispatch.`
      )
    }
    let chain = []

    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args)
    }
    chain = middlewares.map(middleware => middleware(middlewareAPI))
    dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}
```
## reducer enhancers
顾名思义，reducer的“增强剂”。首先我们再回到 createStore 中，第一个参数我们需要定义reducers，不考虑state分割的情况下，我们可能会定义成这样：
```js
function counter(state, action) {
  if (typeof state === 'undefined') {
    return 0
  }

  switch (action.type) {
    case 'INCREMENT':
      return state + 1
    case 'DECREMENT':
      return state - 1
    default:
      return state
  }
}
```
然而这个“模板”是一直从 Redux 文档中流传下来的，但如果我写成这样呢？
```js
function counter(state,action){
  //我啥也不做
}
```
实时证明在写成这样在 createStore 的时候也可以通过的，只不过在`store.getState()`的时候是`undefined`。

所以，个人认为没有什么 `reducer enhancers` 的概念。

### dispatch 的时候是如何调用 reducer 的
其实很简单，直接看源码：
```js
  function dispatch(action) {
    // 各种判断
    try {
      isDispatching = true
      // 正常情况下 currentReducer 就是我们传进去的 reducer
      currentState = currentReducer(currentState, action)
    } finally {
      isDispatching = false
    }

    const listeners = (currentListeners = nextListeners)
    for (let i = 0; i < listeners.length; i++) {
      const listener = listeners[i]
      listener()
    }

    return action
  }
```
上述代码中，我们还看到 dispatch 的时候还会调用 `listener()`，`listeners` 是通过 `subscribe` 方法来订阅的，比如通常我们所做的会把 `render` 方法添加到这里。这样就可以实现数据变化后重新渲染了。