# ReduxSaga-debugger

**tip: 本文适合有一定 redux-saga 使用经验的人阅读，为了阅读体验，以下代码只截取部分核心代码**

## 基本流程

要使用 saga 首先要在 redux 中注册 saga 中间件。

```js
applyMiddleware(createSagaMiddleware());
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

已知 put 一个 action 会首先从 takers（也就是 nextTakers） 中找到与之匹配的 taker 并执行它，那么新的问题又来了，是什么时候往 nextTakers 中赋值的呢？答案就在 channel.take 身上，我在源码中搜索了一下 channel.take，发现它是在 runTakeEffect 中触发的.

我们使用 take 时往往是这样子的

```js
yield take('TODO')
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
