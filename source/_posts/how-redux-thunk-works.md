---
title: How Redux Thunk Works
date: 2018-08-30 10:58:31
tags:
---

Redux thunk has been quite popular as its easiness to handle asynchronous action dispatch in redux store. `Redux Saga` is awesome, but sometimes, especially for a medium or small size project, there is no need for us to involve `Saga` as it requires more consideration and structuring. Moreover, for project in medium scale, it's not worth involving Redux-Saga as learning curve for concepts and usage of generator function and side-effect is much more than Redux Thunk.

> createStore in Redux

````Javascript
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

  ...
}
````

Let's go to key point right now. 

````Javascript
return enhancer(createStore)(reducer, preloadedState)
````

Hence, `enhancer` will be a higher order function. And`createStore`, `reducer` and `preloadState` are passed as parameters.

> When creating redux store

````Javascript
const thunk = createThunkMiddleware()
const store = createStore(reducer, applyMiddleware(thunk))
````
As ya can see, `applyMiddleware(thunk)` will executed like `applyMiddleware(thunk)(createStore)(reducer, preloadedState)`.
So what exactly happens in `applyMiddlewared`? Go deeper.

````Javascript
// applyMiddleware from redux
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
````
Destructuring assignment tells us `thunk` will be passed to **middlewares** in `applyMiddleware` as an array.
After a few steps of execution, we go to the following line:
````Javascript
chain = middlewares.map(middleware => middleware(middlewareAPI))
````
`middlewareAPI` is an Object with `getState` and `dispatch` as its properties. Here, `dispatch` property is different from local variable `dispatch`. The local variable `dispatch` is the real dispatching action from redux-store which we can dispatch action and change the corresponding data structure in redux store. Yet, `dispatch` property refers to a function which accepts `action` as a parameter.
<br>
So one thing left.

> const thunk = createThunkMiddleware()

`thunk` is the middleware we create and pass to `applyMiddleware` as a parameter.
````Javascript
function createThunkMiddleware(extraArgument) {
  return ({ dispatch, getState }) => next => action => {
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }
    return next(action);
  };
}
const thunk = createThunkMiddleware();
thunk.withExtraArgument = createThunkMiddleware;
export default thunk;
````
See, it's a functionality loaded of simplicity and elegance. `thunk` is a function accepts `middlewareAPI` as the result we have already got from our previous walking through on `applyMiddleware`. We have missed one important thing, but I don't wanna put too much elaboration on that right now. Just long story short on the following line:
````Javascript
chain = middlewares.map(middleware => middleware(middlewareAPI))
dispatch = compose(...chain)(store.dispatch)
````
As we decorate our middlewares, we chain'em up by invoking compose function. `compose` is from `redux/compose`. The purpose of this functionality is to chain up all functions and execute them in sequence. But here, there is only one middleware, composing it up will only return this function.
After composing it, as we can see, `store.dispatch` will be passed as middleware's parameter. `store.dispatch` is corresponding to **next** in `createThunkMiddleware`.

Alright, after the aforementioned process is done, we expose a function like:
````Javascript
(action) => {
    if (typeof action === 'function') {
        return action(dispatch, getState, extraArgument);
    }
    return next(action);
}
````
If action is like original Action Object, we directly dispatch them as usual.
If action turns out to be a function, `dispatch`, `getState`, `extraArgument` will be passed into functionality as its three parameters, and do the following steps. You can put your asynchronous logics into your self defined functionality, when your final result is returned from backend, you are allowed to `dispatch` your real action Object.

### Nail it.