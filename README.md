# ReduxSaga-debugger

**tip: 本文适合有一定 redux-saga 使用经验的人阅读，为了阅读体验，以下代码只截取部分核心代码**

## 基本流程

要使用 saga 首先要在 redux 中注册 saga 中间件。并且 调用它的 run 方法注册 rootSaga

```js
const sagaMiddleware = createSagaMiddleware();
applyMiddleware(sagaMiddleware);
sagaMiddleware.run(rootSaga);
```

对应的`createSagaMiddleware`返回值如下：

```js
function sagaMiddleware({ getState, dispatch }) {
  return (next) => (action) => {
    // 命中reducers
    const result = next(action);
    // 往channel中put一个action
    channel.put(action);
    return result;
  };
}
```

根据以上代码可知，外部触发一个 action 之后，如果 reducers 中匹配到对应的 action 则执行下一个中间件，否则往 channel 中 put 此 action，所以我们可以猜测，put 类似于 dispatch，真相如何让我们来一窥究竟

`put` 源码

```js
put(input) {
	const takers = (currentTakers = nextTakers)

	for (let i = 0, len = takers.length; i < len; i++) {
		const taker = takers[i]

		// 如果命中，执行taker
		if (taker[MATCH](input)) {
				taker.cancel()
				taker(input)
		}
	}
}
```

已知 put 一个 action 会首先从 takers（也就是 nextTakers） 中找到与之匹配的 taker 并执行它，那么新的问题又来了，是什么时候往 nextTakers 中赋值的呢？我们可以大胆猜测一下，一定是`sagaMiddleware.run(rootSaga)`这段代码，只有在全局注册好，才能在任意 put 触发的时候匹配上，我们可以来看一下

sagaMiddleware.run 源码

```js
sagaMiddleware.run = (...args) => {
  return boundRunSaga(...args);
};
```

boundRunSaga 其实就是对应源码中的 runSaga,其主要作用是把 rootSaga 转化成 iterator 变量，然后传入 proc 函数中并调用,proc 相当于是事件处理器，每次进来都会调用 next(),而 next 中会计算 iterator.next()的结果传入并调用 digestEffect

```js

```

channel.take 源码

```js
take(cb, matcher = matchers.wildcard) {
	cb[MATCH] = matcher
	//
	ensureCanMutateNextTakers()
	nextTakers.push(cb)

	cb.cancel = once(() => {
		ensureCanMutateNextTakers()
		remove(nextTakers, cb)
	})
}
```

## 设计思路

- 为什么需要 call、put 等 api，这些 api 看起来都是多余的

  [参考链接](https://redux-saga-in-chinese.js.org/docs/basics/DeclarativeEffects.html)
