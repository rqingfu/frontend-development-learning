# redux源码学习
## 序言
使用redux已有一段时间了，redux能为JavaScript app提供可以测的状态容器；具体的使用可参看[此处](http://redux.js.org/)。redux源码对第三方的依赖很少，是学习牛人源码的不错选择，同时其源码注解也很详细。

## 源码结构
reduex的[src]源码组织如下：
```
|--utils
|--applyMiddleware.js
|--bindActionCreators.js
|--combineReducers.js
|--compose.js
|--createStore.js
+--index.js
```
这里index.js是入口文件，具体源码下面分析。
## 源码阅读
### 入口文件index.js
入口文件的主要功能是导出redux的提供的各模块函数。这里比较有意思的是isCrushed函数，其是一个dummy[虚设的]函数，用于确保在非生产环境时使用的代码时非压缩的，便于开发者定位调试。
```javascript
import createStore from './createStore'
import combineReducers from './combineReducers'
import bindActionCreators from './bindActionCreators'
import applyMiddleware from './applyMiddleware'
import compose from './compose'
import warning from './utils/warning'

/*
* 空(dummy)函数，作用是探测其函数名是否被压缩，从而探知代码是否是压缩版[min版]。
* 若该函数已经被修改（即被压缩）并且NODE_ENV !== 'production'[非生产环境]，则提醒用户。
*/
function isCrushed() {}

if (
  process.env.NODE_ENV !== 'production' &&
  typeof isCrushed.name === 'string' &&
  isCrushed.name !== 'isCrushed'
) {
  warning(
    'You are currently using minified code outside of NODE_ENV === \'production\'. ' +
    'This means that you are running a slower development build of Redux. ' +
    'You can use loose-envify (https://github.com/zertosh/loose-envify) for browserify ' +
    'or DefinePlugin for webpack (http://stackoverflow.com/questions/30030031) ' +
    'to ensure you have the correct code for your production build.'
  )
}

export {
  createStore,
  combineReducers,
  bindActionCreators,
  applyMiddleware,
  compose
}
```
### createStore.js 
createStore文件主要提供createStore函数和私有Action Type集。
```javascript
import isPlainObject from 'lodash/isPlainObject'
import $$observable from 'symbol-observable'

/**
 * Redux预留的私有action type集。
 * 对于任意未知actions， 必须返回当前state（current state）。
 * 若当前state为undefined，则必须返回初始state。
 * 开发者在代码中不要直接引用下面这些action类型。
 */
export const ActionTypes = {
  INIT: '@@redux/INIT'
}

/**
 * 函数创建一个Redux store对象，该store对象持有一棵状态树（state tree)。状态树本质上就是一个对象。
 * 改变store对象中的state数据只能通过调用store提供的dispache方法。改变state数据的唯一真确姿势。
 *
 * 每个应用（app）应该有且只有一个store对象。
 * 如果需要明确说明state tree的不同部分对应某类具体action，
 * 可以通过使用combineReducers函数来把不同的reducer归并为一个reducer函数。
 *
 * @param {Function} reducer 参数类型是函数，
 * 该函数接受的参数是当前state tree和要处理的action，返回的是处理完action后的state tree。
 *
 * @param {any} [preloadedState] 可选的任意类型，初始化state。You may optionally specify it
 * to hydrate the state from the server in universal apps, or to restore a
 * previously serialized user session.
 * If you use `combineReducers` to produce the root reducer function, this must be
 * an object with the same shape as `combineReducers` keys.
 *
 * @param {Function} [enhancer] 可选的函数，起作用是增强store对象。
 * 如可以使用中间件，时间流（time travel）及持久化（persistence）等第三方工具提供的功能来增强store对象。
 * 唯一的store enhancer函数是applyMiddleware。
 *
 * @returns {Store} store对象，该对象能让开发者查看state，派发action（dispatch action）和订阅state改变。
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

    return enhancer(createStore)(reducer, preloadedState)
  }

  if (typeof reducer !== 'function') {
    throw new Error('Expected the reducer to be a function.')
  }

  let currentReducer = reducer
  let currentState = preloadedState
  let currentListeners = []
  let nextListeners = currentListeners
  let isDispatching = false

  function ensureCanMutateNextListeners() {
    if (nextListeners === currentListeners) {
      nextListeners = currentListeners.slice()
    }
  }

  /**
   * 读取store管理的当前state树。
   *
   * @returns {any} 任意类型，应用的当前state树。
   */
  function getState() {
    return currentState
  }

  /**
   * 添加一个变化监听函数（change listener）。
   * 当某个action被触发时都会导致该监听函数被调用，并且state树的数据可能会潜在的改变。
   * 在回调中可以调用getState函数来获取当前state树。
   *
   * 在变化监听函数中可能会调用dispatch函数，此时请注意下面的警告：
   *
   * 1. 在dispatch函数调用前订阅被快照（即会备份一份监听函数列表）。
   * 如果在监听函数被调用的同时你订阅或取消订阅，这对正在进行中的dispatch函数没有任何影响。
   * 但是，对于下一次dispatch执行，不论是嵌套执行与否，都将会使用最新的订阅列表快照。
   *
   * 2. 由于在监听函数被调用之前state可能因为嵌套的dispatch被更新多次，
   * 因此监听函数应该不能被期望可以检测到每次的state改变。
   * 然而，可以保证一点，在dispatch开始前所有注册的订阅被调用时使用的state都是最新的。
   *
   * @param {Function} listener 监听函数，每次dispatch后的回调函数。
   * @returns {Function} 函数，用于删除该监听函数。
   */
  function subscribe(listener) {
    if (typeof listener !== 'function') {
      throw new Error('Expected listener to be a function.')
    }

    let isSubscribed = true

    ensureCanMutateNextListeners()
    nextListeners.push(listener)

    return function unsubscribe() {
      if (!isSubscribed) {
        return
      }

      isSubscribed = false

      ensureCanMutateNextListeners()
      const index = nextListeners.indexOf(listener)
      nextListeners.splice(index, 1)
    }
  }

  /**
   * 派发action。是触发state改变的唯一方式。
   *
   * reducer函数将接受当前state和给定的action被调用。
   * 函数返回值将被看作是当前state的下一个state，并且监听函数会被通知。
   *
   * 基础实现仅支持普通对象action（plain object actions）。
   * 如果想要派发Promise，Observable，thunk或其他，需要把createStore函数传入对应的中间件。
   * 例如，查看redux-thunk包文档。使用这种方式，中间件最终还是会派发普通对象action。
   *
   * @param {Object} action 普通对象，表示改变什么。
   * 一个不错的点子是，action系列化以便记录和还原用户session，或者使用时间旅行插件redux-devtool。
   * action必须有type字段且type的值不能为undefined。建议字符串常量作为action的type字段。
   *
   * @returns {Object} 方便起见，返回派发的action。
   *
   * 注意，如果使用自定义中间件，dispatch可能会返回其他值，如Promise。
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
      currentState = currentReducer(currentState, action)
    } finally {
      isDispatching = false
    }

    const listeners = currentListeners = nextListeners
    for (let i = 0; i < listeners.length; i++) {
      const listener = listeners[i]
      listener()
    }

    return action
  }

  /**
   * 替换store使用的当前reducer函数来计算state。
   *
   *如果你实现了代码分离或者想要动态加载reducer函数，可以使用该函数。如果你实现了redux的热加载机制，可以使用该函数。
   *
   * @param {Function} nextReducer store替换使用的reducer函数。
   * @returns {void}
   */
  function replaceReducer(nextReducer) {
    if (typeof nextReducer !== 'function') {
      throw new Error('Expected the nextReducer to be a function.')
    }

    currentReducer = nextReducer
    dispatch({ type: ActionTypes.INIT })
  }

  /**
   * Interoperability point for observable/reactive libraries.
   * @returns {observable} A minimal observable of state changes.
   * For more information, see the observable proposal:
   * https://github.com/tc39/proposal-observable
   */
  function observable() {
    const outerSubscribe = subscribe
    return {
      /**
       * The minimal observable subscription method.
       * @param {Object} observer Any object that can be used as an observer.
       * The observer object should have a `next` method.
       * @returns {subscription} An object with an `unsubscribe` method that can
       * be used to unsubscribe the observable from the store, and prevent further
       * emission of values from the observable.
       */
      subscribe(observer) {
        if (typeof observer !== 'object') {
          throw new TypeError('Expected the observer to be an object.')
        }

        function observeState() {
          if (observer.next) {
            observer.next(getState())
          }
        }

        observeState()
        const unsubscribe = outerSubscribe(observeState)
        return { unsubscribe }
      },

      [$$observable]() {
        return this
      }
    }
  }

  // store创建时，派发`INIT` action,
  // 目的是使每个reducer函数返回它们的初始state,
  // 有效的构建初始state树。
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

### compose.js
```javascript
/**
 * 从右到左构建单参数函数。最右边的函数可以接受多个参数
 * 其为得到的复合函数提供参数
 *
 * @param {...Function} funcs 将要构建的函数
 * @returns {Function} 函数，该函数由参数函数从右到左构成
 * 例如, compose(f, g, h)等同于如下函数(...args) => f(g(h(...args)))。
 */

export default function compose(...funcs) {
  if (funcs.length === 0) {
    return arg => arg
  }

  if (funcs.length === 1) {
    return funcs[0]
  }

  return funcs.reduce((a, b) => (...args) => a(b(...args)))
}
```
### combineReducers.js
```javascript
import { ActionTypes } from './createStore'
import isPlainObject from 'lodash/isPlainObject'
import warning from './utils/warning'

function getUndefinedStateErrorMessage(key, action) {
  const actionType = action && action.type
  const actionName = (actionType && `"${actionType.toString()}"`) || 'an action'

  return (
    `Given action ${actionName}, reducer "${key}" returned undefined. ` +
    `To ignore an action, you must explicitly return the previous state. ` +
    `If you want this reducer to hold no value, you can return null instead of undefined.`
  )
}

function getUnexpectedStateShapeWarningMessage(inputState, reducers, action, unexpectedKeyCache) {
  const reducerKeys = Object.keys(reducers)
  const argumentName = action && action.type === ActionTypes.INIT ?
    'preloadedState argument passed to createStore' :
    'previous state received by the reducer'

  if (reducerKeys.length === 0) {
    return (
      'Store does not have a valid reducer. Make sure the argument passed ' +
      'to combineReducers is an object whose values are reducers.'
    )
  }

  if (!isPlainObject(inputState)) {
    return (
      `The ${argumentName} has unexpected type of "` +
      ({}).toString.call(inputState).match(/\s([a-z|A-Z]+)/)[1] +
      `". Expected argument to be an object with the following ` +
      `keys: "${reducerKeys.join('", "')}"`
    )
  }

  const unexpectedKeys = Object.keys(inputState).filter(key =>
    !reducers.hasOwnProperty(key) &&
    !unexpectedKeyCache[key]
  )

  unexpectedKeys.forEach(key => {
    unexpectedKeyCache[key] = true
  })

  if (unexpectedKeys.length > 0) {
    return (
      `Unexpected ${unexpectedKeys.length > 1 ? 'keys' : 'key'} ` +
      `"${unexpectedKeys.join('", "')}" found in ${argumentName}. ` +
      `Expected to find one of the known reducer keys instead: ` +
      `"${reducerKeys.join('", "')}". Unexpected keys will be ignored.`
    )
  }
}

function assertReducerShape(reducers) {
  Object.keys(reducers).forEach(key => {
    const reducer = reducers[key]
    const initialState = reducer(undefined, { type: ActionTypes.INIT })

    if (typeof initialState === 'undefined') {
      throw new Error(
        `Reducer "${key}" returned undefined during initialization. ` +
        `If the state passed to the reducer is undefined, you must ` +
        `explicitly return the initial state. The initial state may ` +
        `not be undefined. If you don't want to set a value for this reducer, ` +
        `you can use null instead of undefined.`
      )
    }

    const type = '@@redux/PROBE_UNKNOWN_ACTION_' + Math.random().toString(36).substring(7).split('').join('.')
    if (typeof reducer(undefined, { type }) === 'undefined') {
      throw new Error(
        `Reducer "${key}" returned undefined when probed with a random type. ` +
        `Don't try to handle ${ActionTypes.INIT} or other actions in "redux/*" ` +
        `namespace. They are considered private. Instead, you must return the ` +
        `current state for any unknown actions, unless it is undefined, ` +
        `in which case you must return the initial state, regardless of the ` +
        `action type. The initial state may not be undefined, but can be null.`
      )
    }
  })
}

/**
 * 将一个其值是不同的reducer函数的对象转化为单个reducer函数。
 * 该函数会调用对象中的每一个reducer，并且把得到的结果合并为单一的state对象，
 * 这些对象的key对应的是传入对象的该reducer函数对应的key。
 *
 * @param {Object} reducers object对象，其key对应的值是不同的对象。
 * 根据该对象合并为单一的reducer函数。
 * 获取该对象的一种简便方式是通过es6提供的import * as reducers语法。
 * 对象中的reducers函数对任意action都不能返回undefined。
 * 另外，如果传给reducer函数的state是undefined，其应该返回初始state。
 * 对任何不能识别的action且state不为undefined，则返回给定的传入的state。
 *
 * @returns {Function} reducer函数，其会调用上面reducers中的每个reducer函数，
 * 并且构建一个和reducers对象key相同的state对象。
 */
export default function combineReducers(reducers) {
  const reducerKeys = Object.keys(reducers)
  const finalReducers = {}
  for (let i = 0; i < reducerKeys.length; i++) {
    const key = reducerKeys[i]

    if (process.env.NODE_ENV !== 'production') {
      if (typeof reducers[key] === 'undefined') {
        warning(`No reducer provided for key "${key}"`)
      }
    }

    if (typeof reducers[key] === 'function') {
      finalReducers[key] = reducers[key]
    }
  }
  const finalReducerKeys = Object.keys(finalReducers)

  let unexpectedKeyCache
  if (process.env.NODE_ENV !== 'production') {
    unexpectedKeyCache = {}
  }

  let shapeAssertionError
  try {
    assertReducerShape(finalReducers)
  } catch (e) {
    shapeAssertionError = e
  }

  return function combination(state = {}, action) {
    if (shapeAssertionError) {
      throw shapeAssertionError
    }

    if (process.env.NODE_ENV !== 'production') {
      const warningMessage = getUnexpectedStateShapeWarningMessage(state, finalReducers, action, unexpectedKeyCache)
      if (warningMessage) {
        warning(warningMessage)
      }
    }

    let hasChanged = false
    const nextState = {}
    for (let i = 0; i < finalReducerKeys.length; i++) {
      const key = finalReducerKeys[i]
      const reducer = finalReducers[key]
      const previousStateForKey = state[key]
      const nextStateForKey = reducer(previousStateForKey, action)
      if (typeof nextStateForKey === 'undefined') {
        const errorMessage = getUndefinedStateErrorMessage(key, action)
        throw new Error(errorMessage)
      }
      nextState[key] = nextStateForKey
      hasChanged = hasChanged || nextStateForKey !== previousStateForKey
    }
    return hasChanged ? nextState : state
  }
}
```
### bindActionCreators.js
```javascript
import warning from './utils/warning'

function bindActionCreator(actionCreator, dispatch) {
  return (...args) => dispatch(actionCreator(...args))
}

/**
 * 把值是action creator函数的对象转化为同样值的对象，
 * 但是每个函数被包装到dispatch调用，方便被直接调用。
 * 其仅仅是一个方便函数，类似于调用。
 * `store.dispatch(MyActionCreators.doSomething())`。
 *
 * 方便起见，也可以给第一个参数传递为函数并且得到一个函数作为返回。
 *
 * @param {Function|Object} actionCreators 函数或对象。
 * 若是对象，其值是action creator函数。一种简便的获取方式是通过es6的import * as语法。
 * 也可以传递一个函数作为参数。
 *
 * @param {Function} dispatch Redux store可访问的`dispatch`函数。
 *
 * @returns {Function|Object} 返回和形参类似的对象，
 * 但每个action creator函数被包裹到dispatch调用中。
 * 如果形参传入的是action creator函数，则返回的也将是一个单一函数。
 */
export default function bindActionCreators(actionCreators, dispatch) {
  if (typeof actionCreators === 'function') {
    return bindActionCreator(actionCreators, dispatch)
  }

  if (typeof actionCreators !== 'object' || actionCreators === null) {
    throw new Error(
      `bindActionCreators expected an object or a function, instead received ${actionCreators === null ? 'null' : typeof actionCreators}. ` +
      `Did you write "import ActionCreators from" instead of "import * as ActionCreators from"?`
    )
  }

  const keys = Object.keys(actionCreators)
  const boundActionCreators = {}
  for (let i = 0; i < keys.length; i++) {
    const key = keys[i]
    const actionCreator = actionCreators[key]
    if (typeof actionCreator === 'function') {
      boundActionCreators[key] = bindActionCreator(actionCreator, dispatch)
    } else {
      warning(`bindActionCreators expected a function actionCreator for key '${key}', instead received type '${typeof actionCreator}'.`)
    }
  }
  return boundActionCreators
}
```
### applyMiddleware.js
```javascript
import compose from './compose'

/**
 * 创建store enhancer函数,该enhancer函数应用中间件到redux store的dispatch函数。
 * 这对于一些任务是方便的，比如以精简的方式表达异步action，或者以日志的方式记录每个action。
 *
 * 看`redux-thunk`包做为Redux中间件的例子。
 *
 * 因为中间件可能是异步的，所以在构建链中其是第一个store enhancer函数
 * 
 * 注意每个中间件将接受dispatch和各getState函数作为参数名
 * 
 *
 * @param {...Function} middlewares 将被应用的中间件链。
 * @returns {Function} 使用中间件的store enhancer函数。
 */
export default function applyMiddleware(...middlewares) {
  return (createStore) => (reducer, preloadedState, enhancer) => {
    const store = createStore(reducer, preloadedState, enhancer)
    let dispatch = store.dispatch
    let chain = []

    const middlewareAPI = {
      getState: store.getState,
      dispatch: (action) => dispatch(action)
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