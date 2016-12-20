# Redux 源码浅析


redux的源码目录
```
├── utils/
│     ├── warning.js 
├── applyMiddleware.js
├── bindActionCreators.js
├── combineReducers.js
├── compose.js
├── createStore.js
├── index.js # 入口文件
```

除去 `utils/warning.js` 以及入口文件 `index.js`，剩下那 5 个就是 Redux 的 API

##  compose(...functions)
> 这是一个纯函数 

### 源码分析

```js
/**
 * 实际运用是这样子的：
 * compose(f, g, h)(...arg) => f(g(h(...args)))
 *
 * 这里面它用到了 reduceRight，因此执行顺序是从右到左
 *`reduceRight` 可传入初始值：
 * @param  {多个函数，用逗号隔开}
 * @return {函数}
 */
// ...funcs写法是es6的rest参数 ，我一开始没看懂，翻了下es6才知道
export default function compose(...funcs) {
  /**
  *...funcs 对应的es5写法 把arguments转化为一个数组
  *  for (var _len = arguments.length, funcs = Array(_len), _key = 0; _key < _len; _key++) {
  *    funcs[_key] = arguments[_key];
  *  }
  */
  if (funcs.length === 0) {
    return arg => arg
  }

  if (funcs.length === 1) {
    return funcs[0]
  }

  const last = funcs[funcs.length - 1]
  const rest = funcs.slice(0, -1)
  return (...args) => rest.reduceRight((composed, f) => f(composed), last(...args))
}
```

```js
// 由于 reduce 与 reduceRight方法 仅仅是方向的不同，下面是reduce使用的一个例子
var arr = [1, 2, 3, 4, 5]

var result1 = arr.reduce(function(num, i) {
  return num + i
})
console.log(result1) // 15

var result2 = arr.reduce(function(num, i) {
  return num + i
}, 10) // 传入一个初始值
console.log(result2) // 25
```

```html
<!DOCTYPE html>
<html>
<head>
  <script src="//cdn.bootcss.com/redux/3.5.2/redux.min.js"></script>
</head>
<body>
<script>
function func1(num) {
  console.log('func1 获得参数 ' + num);
  return num + 1;
}

function func2(num) {
  console.log('func2 获得参数 ' + num);
  return num + 2;
}
  
function func3(num) {
  console.log('func3 获得参数 ' + num);
  return num + 3;
}

// 普通写法
var re1 = func3(func2(func1(0)));
console.log('re1：' + re1);

console.log('===============');

// compose
var re2 = Redux.compose(func3, func2, func1)(0);
console.log('re2：' + re2);
</script>
</body>
</html>
```

控制台输出：

```
func1 获得参数 0
func2 获得参数 1
func3 获得参数 3
re1：6
===============
func1 获得参数 0
func2 获得参数 1
func3 获得参数 3
re2：6
```

##  createStore(reducer, initialState, enhancer)
###  源码分析

```js
import isPlainObject from 'lodash/isPlainObject'
import $$observable from 'symbol-observable'

/**
 * 这是 Redux 的私有 action 常量
 * 初始化的时候会调用
 */
export var ActionTypes = {
  INIT: '@@redux/INIT'
}

/**
 * @param  {函数}  reducer
 * @param  {对象}  preloadedState 初始化state树
 * @param  {函数}  enhancer 是做store增强器用的。具体我也呵呵了
 *
 * @return {Store}
 */
export default function createStore(reducer, preloadedState, enhancer) {

  if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
    enhancer = preloadedState
    preloadedState = undefined
  }

  if (typeof enhancer !== 'undefined') {
    if (typeof enhancer !== 'function') {
      throw new Error('Expected the enhancer to be a function.')
    }
    // 存在 enhancer 就立即执行，返回增强版的 createStore
    return enhancer(createStore)(reducer, preloadedState)
  }

  if (typeof reducer !== 'function') {
    throw new Error('Expected the reducer to be a function.')
  }
  
  var currentReducer = reducer
  var currentState = preloadedState //     这就是整个应用的 state
  var currentListeners = [] //             用于存储订阅的回调函数，dispatch 后逐个执行
  var nextListeners = currentListeners 
  var isDispatching = false

  
  function ensureCanMutateNextListeners() {
    if (nextListeners === currentListeners) {
      nextListeners = currentListeners.slice()
    }
  }

  /**
   * 返回 state
   */
  function getState() {
    return currentState
  }

  /**
   * 负责注册回调函数
   * 
   * 这里需要注意的就是，回调函数中如果需要获取 state
   * 那每次获取都请使用 getState()，而不是开头用一个变量缓存住它
   * 因为回调函数执行期间，有可能有连续几个 dispatch 让 state 改得物是人非
   * 而且别忘了，dispatch 之后，整个 state 是被完全替换掉的
   * 你缓存的 state 指向的可能已经是老掉牙的 state 了！！！
   *
   * @param  {函数} 想要订阅的回调函数
   * @return {函数} 取消订阅的函数
   */
  function subscribe(listener) {
    if (typeof listener !== 'function') {
      throw new Error('Expected listener to be a function.')
    }

    var isSubscribed = true

    ensureCanMutateNextListeners() 
    nextListeners.push(listener)   // 新增订阅在 nextListeners 中操作

    // 返回一个取消订阅的函数
    return function unsubscribe() {
      if (!isSubscribed) {
        return
      }

      isSubscribed = false

      ensureCanMutateNextListeners() 
      var index = nextListeners.indexOf(listener)
      nextListeners.splice(index, 1) // 取消订阅还是在 nextListeners 中操作
    }
  }

  /**
   * 改变应用状态 state 的唯一途径：dispatch 一个 action
   * 内部的实现是：往 reducer 中传入 currentState 以及 action
   * 用其返回值替换 currentState，最后逐个触发回调函数
   *
   * 如果 dispatch 的不是一个对象类型的 action（同步的），而是 Promise （异步的）
   * 则需引入 redux-thunk 等中间件来处理action
   * 
   * @param {对象} action
   * @return {对象} action
   */
  function dispatch(action) {
    if (!isPlainObject(action)) {
      throw new Error(
        'Actions must be plain objects. ' +
        'Use custom middleware for async actions.'
      )
    }

    if (typeof action.type === 'undefined') {
      throw new Error(
        'Actions may not have an undefined "type" property. ' +
        'Have you misspelled a constant?'
      )
    }

    if (isDispatching) {
      throw new Error('Reducers may not dispatch actions.')
    }

    try {
      isDispatching = true
      // 关键点：currentState 与 action 会流通到所有的 reducer
      // 所有 reducer 的返回值整合后，替换掉当前的 currentState
      currentState = currentReducer(currentState, action)
    } finally {
      isDispatching = false
    }

    var listeners = currentListeners = nextListeners

    // 逐个触发回调函数
    for (var i = 0; i < listeners.length; i++) {
      listeners[i]()

      var listener = listeners[i]
      listener()
    }

    return action // 为了方便链式调用，dispatch 执行完毕后，返回 action
  }

  /**
   * 替换当前 reducer
   * 主要用于代码分离按需加载、热替换;貌似需要webpack来配置
   *
   * @param {函数} nextReducer
   */
  function replaceReducer(nextReducer) {
    if (typeof nextReducer !== 'function') {
      throw new Error('Expected the nextReducer to be a function.')
    }

    currentReducer = nextReducer         // 新的reducer直接覆盖老的reducer！
    dispatch({ type: ActionTypes.INIT }) // 触发生成新的 state 树
  }



  // 在store对象创建的时候，这里会主动调用dispatch({ type: ActionTypes.INIT });
  // 来对内部状态进行初始化。通过断点或者日志打印就可以看到，store对象创建的同时，reducer就会被调用进行初始化。
  dispatch({ type: ActionTypes.INIT })

  return {
    dispatch,
    subscribe,
    getState,
    replaceReducer,
    [$$observable]: observable
  }
}

```


## combineReducers(reducers)
### 应用场景


```js
/**  code-7 **/
var initState = {
  counter: 0,
  todos: []
}

function reducer(state = initState, action) {

  switch (action.type) {
    case 'ADD_TODO':

      state.todos.push(action.payload)
      return state

    default:
      return state
  }
}
```

上面的 `reducer` 仅仅是实现了 “新增待办事项” 的 `state` 的处理  
我们还有计数器的功能，下面我们继续增加计数器 “增加 1” 的功能：

```js
/**  code-8 **/
var initState = { counter: 0, todos: [] }

function reducer(state = initState, action) {

  switch (action.type) {
    case 'ADD_TODO': // 新增待办事项
      state.todos.push(action.payload)
      return state;
      break   
    case 'INCREMENT': // 计数器加 1
      state.counter = state.counter + 1
      return state;
      break
  }
  return state
}
```

如果说还有其他的动作，都需要在 `code-8` 这个 `reducer` 中继续堆砌处理逻辑  
但我们知道，计数器 与 待办事项 属于两个不同的模块，不应该都堆在一起写  
如果之后又要引入新的模块（例如留言板），该 `reducer` 会越来越臃肿  
此时就是 `combineReducers` 派上用场的时候：



```js
/**  code-9 **/
/* reducers/index.js */
import { combineReducers } from 'redux'
import counterReducer from './counterReducer'
import todosReducer from './todosReducer'

const rootReducer = combineReducers({
  counter: counterReducer, // <-------- 键名就是该 reducer 对应管理的 state
  todos: todosReducer
})

export default rootReducer

-------------------------------------------------

/* reducers/counterReducer.js */
export default function counterReducer(state = initState, action) { // 传入的 state 其实是 state.counter
  switch (action.type) {
    case 'INCREMENT':
       state.counter = state.counter + 1
       return state;
    default:
      return state
  }
}

-------------------------------------------------

/* reducers/todosReducers */
export default function todosReducer(state = initState, action) { // 传入的 state 其实是 state.todos
  switch (action.type) {
    case 'ADD_TODO':
      return [ ...state.todos, action.payload ]
    default:
      return state
  }
}
```

`code-8 reducer` 与 `code-9 rootReducer` 的功能是一样的，但后者的各个子 `reducer` 仅维护对应的那部分 `state`  
这样可维护性、可扩展性好多了

> 而 Redux 只允许应用中有唯一的 `store`，通过拆分出多个 `reducer` 分别管理对应的 `state`

无论您的应用状态树有多么的复杂，都可以通过逐层下分管理对应部分的 `state`：

> 无论是 `dispatch` 哪个 `action`，都会流通所有的 `reducer`
> 这也是为何 `reducer` 必须返回其对应的 `state` 的原因。否则整合state树时，该 `reducer` 对应的键值就是 `undefined`

### 源码分析

```js
function combineReducers(reducers) {
  var reducerKeys = Object.keys(reducers)
  var finalReducers = {}
  
  for (var i = 0; i < reducerKeys.length; i++) {
    var key = reducerKeys[i]
    if (typeof reducers[key] === 'function') {
      finalReducers[key] = reducers[key]
    }
  }

  var finalReducerKeys = Object.keys(finalReducers)

  // 返回合成后的 reducer
  return function combination(state = {}, action) {
    var hasChanged = false
    var nextState = {}
    for (var i = 0; i < finalReducerKeys.length; i++) {
      var key = finalReducerKeys[i]
      var reducer = finalReducers[key]
      var previousStateForKey = state[key]                       // 获取当前子 state
      var nextStateForKey = reducer(previousStateForKey, action) // 执行各子 reducer 中获取子 nextState
      nextState[key] = nextStateForKey                           // 将子 nextState 赋值给 对应的 reducek key
      hasChanged = hasChanged || nextStateForKey !== previousStateForKey
    }
    return hasChanged ? nextState : state
  }
}

```


## [bindActionCreators(actionCreators, dispatch)][bindActionCreators]
> 这个API是最简单的， 给action添加dispatch功能

###  源码分析

```js
/* 给Action加上自动 dispatch功能 */
function bindActionCreator(actionCreator, dispatch) {
  return (...args) => dispatch(actionCreator(...args))
}

export default function bindActionCreators(actionCreators, dispatch) {

  var keys = Object.keys(actionCreators)
  var boundActionCreators = {}
  // 给多个Action添加 dispatch
  for (var i = 0; i < keys.length; i++) {
    var key = keys[i]
    var actionCreator = actionCreators[key]
    if (typeof actionCreator === 'function') {

      boundActionCreators[key] = bindActionCreator(actionCreator, dispatch)
    }
  }
  return boundActionCreators
}
```

### 应用场景

```html
<--!  code-10 -->

<script>
var action = addTodo('action') // 执行 Action Creator 获得 action
  store.dispatch(action) // 手动显式 dispatch 一个 action
</script>
```

我们看到，调用 `addTodo` 这个 Action Creator 后得到一个 `action`，之后又要手动 `dispatch(action)`  
如果是只有一个两个 Action Creator 还是可以接受，但如果有很多个那就显得有点重复了
这个时候我们就可以利用 `bindActionCreators` 实现自动 `dispatch`：

```html

<script>
// 全局引入 Redux，同时 store 是全局变量
var actionsCreators = Redux.bindActionCreators(
  { addTodo: addTodo },
  store.dispatch // 传入 dispatch 函数
)
 actionCreators.addTodo('action')// 它会自动 dispatch

</script>
```


## applyMiddleware(...middlewares)


### Middleware
Redux的中间件机制，其作用就是在 `dispatch` 前后，统一“一些事情”。。。
比如统一的日志记录、引入 thunk 统一处理异步 action等都属于中间件
下面是一个简单的打印动作前后 `state` 的中间件：

```js
/* es6写法 */
const printStateMiddleware = (creatStore) => next => action => {
  console.log('state before dispatch', creatStore.getState())
  
  let returnValue = next(action)

  console.log('state after dispatch', creatStore.getState())

  return returnValue
}

-------------------------------------------------

/* es5写法 */
function printStateMiddleware(creatStore) { // 传入中间件的store
  return function (dispatch) {                 // 传入上级中间件处理逻辑（若无则为原 store.dispatch）

    // 整个函数将会被传到下级中间件（如果有的话）作为它的 dispatch 参数
    return function (action) { // 中间件逻辑处理
      console.log('state before dispatch', creatStore.getState())
  
      var returnValue = dispatch(action) //dispatch 的返回值其实还是 action
  
      console.log('state after dispatch', creatStore.getState())

      return returnValue // 继续传给下一个中间件作为参数 action
    }
  }
}
```
源码分析
```js
import compose from './compose'

/* 传入一系列中间件 */
export default function applyMiddleware(...middlewares) {

  /* 传入 createStore */
  return function(createStore) {
  
    /* 返回一个函数签名跟 createStore 一模一样的函数，亦即返回的是一个增强版的 createStore */
    return function(reducer, preloadedState, enhancer) {
    
      // 用原 createStore 先生成一个 store，其包含 getState / dispatch / subscribe / replaceReducer 四个 API
      var store = createStore(reducer, preloadedState, enhancer)
      
      var dispatch = store.dispatch // 指向原 dispatch
      var chain = [] // 存储中间件的数组
  
      // 提供给中间件的 API（其实都是 store 的 API）
      var middlewareAPI = {
        getState: store.getState,
        dispatch: (action) => dispatch(action)
      }
      
      // 给中间件store 的API，
      chain = middlewares.map(middleware => middleware(middlewareAPI))
      
      // 串联所有中间件
      dispatch = compose(...chain)(store.dispatch)
      // 例如，chain 为 [M3, M2, M1]，而 compose 是从右到左进行“包裹”的
      // 那么，M1 的 dispatch 参数为 store.dispatch（
      // 往后，M2 的 dispatch 参数为 M1 的中间件处理逻辑
      // 同样，M3 的 dispatch 参数为 M2 的中间件处理逻辑
      // 最后，我们得到串联后的中间件链：M3(M2(M1(store.dispatch)))

      return {
        ...store, // store 的 API 中保留 getState / subsribe / replaceReducer
        dispatch  // 新 dispatch 覆盖原 dispatch，往后调用 dispatch 就会触发 chain 内的中间件链式串联执行
      }
    }
  }
}

```

最终返回的虽然还是 `store` 的那四个 API，但其中的 `dispatch` 函数的功能被增强了

###  综合应用
```html
<!DOCTYPE html>
<html>
<head>
  <script src="//cdn.bootcss.com/redux/3.5.2/redux.min.js"></script>
</head>
<body>
<script>
/** Action Creators */
function inc() {
  return { type: 'INCREMENT' };
}
function dec() {
  return { type: 'DECREMENT' };
}

function reducer(state, action) {
  state = state || { counter: 0 };

  switch (action.type) {
    case 'INCREMENT':
      return { counter: state.counter + 1 };
    case 'DECREMENT':
      return { counter: state.counter - 1 };
    default:
      return state;
  }
}

function printStateMiddleware(middlewareAPI) {
  return function (dispatch) {
    return function (action) {
      console.log('dispatch 前：', middlewareAPI.getState());
      var returnValue = dispatch(action);
      console.log('dispatch 后：', middlewareAPI.getState());
      return returnValue;
    };
  };
}

var enhancedCreateStore = Redux.applyMiddleware(printStateMiddleware)(Redux.createStore);
var store = enhancedCreateStore(reducer);

store.dispatch(inc());
store.dispatch(inc());
store.dispatch(dec());
</script>
</body>
</html>
```

控制台输出：

```
dispatch 前：{ counter: 0 }
dispatch 后：{ counter: 1 }

dispatch 前：{ counter: 1 }
dispatch 后：{ counter: 2 }

dispatch 前：{ counter: 2 }
dispatch 后：{ counter: 1 }
```

***


如果有多个中间件以及多个增强器，还可以这样写（请留意序号顺序）：

> 重温一下 `createStore` 完整的函数签名：`function createStore(reducer, preloadedState, enhancer)`

```js
/**  code-11 **/
import { createStore, applyMiddleware, compose } from 'redux'

const store = createStore(
  reducer,
  preloadedState, // 可选，初始化state树的数据
  compose( // compose 是从右到左的！
    applyMiddleware( //  applyMiddleware也是 Store Enhancer ！但这是关乎中间件的增强器，必须置于 compose 执行链的最后
      middleware1,
      middleware2,
      middleware3
    ),
    enhancer3,
    enhancer2,
    enhancer1
  )
)
```




## 总结
Redux 有五个 API，分别是：

* `createStore(reducer, [initialState])`
* `combineReducers(reducers)`
* `applyMiddleware(...middlewares)`
* `bindActionCreators(actionCreators, dispatch)`
* `compose(...functions)`

`createStore` 生成的 `store` 有四个 API，分别是：

* `getState()`
* `dispatch(action)`
* `subscribe(listener)`
* `replaceReducer(nextReducer)`


