## Preact 是如何实现 Hooks 的？

之前比较详细地分享过一次 React Hooks 的基本使用方法以及一些使用中需要注意的细节。

现在，这篇文章主要分享 Preact 是怎么实现 Hooks 的。

> 为什么是 Preact？因为其实现简单，根据源码分析实现逻辑的过程会更加简单。

### 一 Preact 简介

根据 [Preact 官方介绍](https://preactjs.com/)，Preact 就是作为一个轻量化的 React 的替代品而存在。

它与 React 拥有同样的 API，但是更加轻量，设计上也更加简化。如果要分析一些细节上的实现，不妨从 Preact 入手。

不久前看到有些公众号在推送说某个外国企业将应用从 React 迁移到了 Preact（如下图），说明这个类 React 框架也是经受住了很多考验的，其源码具有分析价值。

![img](https://static001.geekbang.org/infoq/20/20f3b3fb2ca75b7b73d6469d870656c1.png)

更多具体的 React 和 Preact 的对比，可以参考知乎的这个问题：[如何看待 React 的替代框架 Preact？](https://www.zhihu.com/question/65479147)。

### 二 提出问题

以下是本文旨在解决的问题：

- **Hooks 的基本实现（包括数据结构）是怎么样的？**
- **Preact 如何区分 renderCallbacks 和 pendingEffects？**
- **为什么 Hooks 要求每次 render 中 Hook 运行的顺序和数量必须一致？**
- **为什么有些 Hooks（useState…）可以触发 re-render？**
- **如何限制开发者使用 Hooks 的时机？**

> 解决问题的方式：阅读 Preact@10.5.13 的源代码。

### 三 具体分析

#### 3.1 基本概念

Preact 中有以下基本概念需要知悉：

- **Component**
  Component 即为组件，可以是 Function Component 也可以是 Class Component，__*在 Preact 中最后都会转换成 Class Component*__。
  Component 是开发者抽象前端 UI 的基本单元。

  一个 Component 定义了一个前端组件的具体组成结构、样式和逻辑。

- **VNode**
  虚拟 DOM，由 Class Component 实例的 render 方法或者 Function Component 通过调用 createElement 函数（或者叫 h 函数，与 JSX 相关）所产生。

  VNode 包含一个前端组件在某个时间节点上的具体信息，它是 DOM 的一种抽象数据结构，可以被渲染成真实 DOM。

  可以说 VNode 是 Component 和 DOM 的中间桥梁。

- **Component 实例**

  Component 是可复用的，所以我们可能会在页面上的不同地方渲染同一个 Component（比如在 HeaderBar 和 SideBar 中都是用了 Icon 组件）。

  ***这些不同地方的组件渲染之后可能会有不同的状态和表现，所以必须针对每一处 Component 实例化，用以维护所有同类组件各自的状态。这一点非常关键，因为 Preact Hooks 就是通过将状态存储在 Component 实例上来实现的。***

  Preact 在渲染页面时调用我们的 Component 类（函数组件会转换成 Class）构造 Component 实例。

- **Option Hooks**

  Preact 官方文档中有一个关于 [Option Hooks](https://preactjs.com/guide/v10/options) 的介绍，在源码中也可以轻松找到其实现，这是一个简单的 JS 对象。它上面挂载了许多的函数，比如 `vnode`、`unmount` 等等公开的函数，还有一些内部定义的函数比如 `_diff`、`_render` 等等。

  这些函数会在 Preact 渲染的不同阶段执行，比如 `_diff` 函数在 diff 阶段之前执行，`_render` 函数在 render 阶段之前执行。

  这些钩子函数的存在使得 Preact 框架的扩展性非常强，可以很容易的实现一些 Preact 插件来干涉其工作流程。

  在内部实现中也非常依赖 Option Hooks，后续分析中还可以见到它的身影。

#### 3.2 函数组件

有了 3.1 小节的知识准备之后，就可以看一下 Preact 中的函数组件是怎么被渲染到页面上的了。

如下图所示：

![](https://tva1.sinaimg.cn/large/e6c9d24egy1gzk0q2r1r7j210h0u0gnn.jpg)

函数组件 Icon 在 mount 阶段会被实例化，但是其实例化方式仍然是通过 Preact.Component 实现，函数组件本身会被当成 render 函数用于生成 VNode。

针对 JSX 中的每一处 `<Icon ... />`，都会实例化一个 Icon 对象。在 mount 完成之后，用户的交互事件或者应用的其他定时任务可能会触发一些实例的 re-render，这时候 Component 实例是复用的。组件内部状态的变更（比如 useState 的数据）就存储在 Component 实例上。

那函数组件的状态具体是怎么存储在 Component 实例上的呢？

#### 3.3 数据结构

在 Component 实例上，有一个名为 `__hooks` 的属性，这个属性用于存储当前组件实例使用到的所有 Hooks 的状态。

Hooks 的基本数据结构（结合下方解释看）：

```typescript
export interface ComponentHooks {
	/** The list of hooks a component uses */
	_list: HookState[];
	/** List of Effects to be invoked after the next frame is rendered */
	_pendingEffects: EffectHookState[];
}

export interface EffectHookState {
	_value?: Effect;
	_args?: any[];
	_cleanup?: Cleanup | void;
}

export type HookState =
	| EffectHookState
	| MemoHookState
	| ReducerHookState
	| ContextHookState
	| ErrorBoundaryHookState;

export type Effect = () => void | Cleanup;
export type Cleanup = () => void;

export interface EffectHookState {
	_value?: Effect;
	_args?: any[];
	_cleanup?: Cleanup | void;
}

export interface MemoHookState {
	_value?: any;
	_args?: any[];
	_factory?: () => any;
}

export interface ReducerHookState {
	_value?: any;
	_component?: Component;
	_reducer?: Reducer<any, any>;
}

export interface ContextHookState {
	/** Whether this hooks as subscribed to updates yet */
	_value?: boolean;
	_context?: PreactContext;
}

export interface ErrorBoundaryHookState {
	_value?: (error: any) => void;
}
```

上方类型中，`ComponentHooks` 即为 `comp.__hooks` 的类型。

`_list` 是一个数组，用于存储多个 hooks 的状态（一个函数组件中可能用到多个 Hook）。数组项的类型对于不同的 Hook 而言是不用的，因为需要存储的数据不一样。

`EffectHook` 用于存储 `useEffect` 的状态。其中 `_value` 是 `useEffect` 接收的第一个参数，`_args` 是 `useEffect` 接收的第二个参数 ，`_cleanup` 是 `useEffect` 接收的第一个参数执行后返回的函数，在 re-render 之前执行。

`MemoHookState` 用于存储 `useMemo` 的状态。其中 `_value` 是缓存的数据，`_args` 是 `useMemo` 接收的第二个参数，`_factory` 是 `useMemo` 接收的第一个参数（即数据的计算函数）。

`ReducerHookState` 是 `useReducer` 的状态。其中 `_value` 是存储的状态（state）和更新方法（dispatch），`_component` 是 Hook 所属的组件实例（用于区分是否是初次渲染），`_reducer` 是 `useReducer` 接收的第一个参数（reducer）。

`ContextHookState` 是 `useContext` 的状态。其中 `_value` 是表示当前组件是否有订阅 Context Provider 的变量，`_context` 是 `useContext` 接收的参数，是一个 context 对象。

以上是各个内置 Hooks 的数据存储结构，那 Hooks 是怎么使用这些数据的呢？

#### 3.4 实现方式

##### 3.4.1 getHookState

这是一个 Preact 的内部工具方法，用于在 Hooks 执行的时候获取当前 Hook 对应的状态。

我们熟知的 `useMemo`, `useReducer` 等 Hooks 在运行时都会先调用 `getHookState` 方法获取对应的状态对象。

实现方式：

```js
function getHookState(index, type) {
	if (options._hook) {
		options._hook(currentComponent, index, currentHook || type);
	}
	currentHook = 0;

	// Largely inspired by:
	// * https://github.com/michael-klein/funcy.js/blob/f6be73468e6ec46b0ff5aa3cc4c9baf72a29025a/src/hooks/core_hooks.mjs
	// * https://github.com/michael-klein/funcy.js/blob/650beaa58c43c33a74820a3c98b3c7079cf2e333/src/renderer.mjs
	// Other implementations to look at:
	// * https://codesandbox.io/s/mnox05qp8
	const hooks =
		currentComponent.__hooks ||
		(currentComponent.__hooks = {
			_list: [],
			_pendingEffects: []
		});

	if (index >= hooks._list.length) {
		hooks._list.push({});
	}
	return hooks._list[index];
}
```

其实现很简单，根据传入的第一个参数（一个索引）在 Component 实例的 `__hooks` 数组（见 3.3 节）上获取对应的状态。

那这个索引是哪里来的呢？

在 Preact 源码中，有一个模块内的全局变量：`currentIndex`。这是一个特殊的变量，它会在 render 开始之前设置为 0（通过 `options._render` 钩子实现），然后每次调用 `getHookState` 之后都会自增 1。

这样就确保了每次运行一个需要存储状态的 Hook（比如 useState）都会得到一个表示其在当前函数组件内执行顺序的序号索引，而 render 之前的清零操作确保了同一个函数组件多次 render 时，内部的每个 Hook 获取到的序号索引是不变的。

*__这就是为什么 Hooks 的使用需要严格注意执行顺序和数量在每次 render 中不变，因为一旦数量变化或者顺序变化，都会导致 Hook 获取到的状态不对，进而导致 bug 的产生。__*

代码中还有一个 `options._hook` 钩子的调用，这个钩子会在 debug 模式下被挂载。在这个钩子中会检查执行时是否能够获取到对应的 Component 实例以及是否允许运行 Hooks：

1. 在 `options.diffed` 钩子中会将 `hooksAllowed` 变量设为 `false`，这样在 diff 完成之后的时间内，是不允许执行 Hooks 的。
2. 在 `options._diff` 钩子中会将 Hooks 模块内的全局变量 `currentComponent` 设为 `null`，这样在 diff 阶段之前也是不允许执行 Hooks 的。

如果在 diff 阶段前后执行 Hooks 会抛出错误：`Hook can only be invoked from render methods.`。

##### 3.4.2 useReducer

`useReducer` 接收一个 `reducer` 函数，一个 `initialState` 和  `init` 方法。

其实现如下：

```js
export function useReducer(reducer, initialState, init) {
	/** @type {import('./internal').ReducerHookState} */
	const hookState = getHookState(currentIndex++, 2);
	hookState._reducer = reducer;
	if (!hookState._component) {
		hookState._value = [
			!init ? invokeOrReturn(undefined, initialState) : init(initialState),

			action => {
				const nextValue = hookState._reducer(hookState._value[0], action);
				if (hookState._value[0] !== nextValue) {
					hookState._value = [nextValue, hookState._value[1]];
					hookState._component.setState({});
				}
			}
		];

		hookState._component = currentComponent;
	}

	return hookState._value;
}

function invokeOrReturn(arg, f) {
	return typeof f == 'function' ? f(arg) : f;
}
```

前文交代过 `_component` 这个属性用来存储 Hook 所属的 Component 实例，当这个属性不为空时说明 Hook 是在组件的 update 阶段执行的，这时候就不需要执行初始化逻辑了。

而初始化逻辑比较简单，通过 `initialState` 和 `init` 参数计算初始状态，`dispatch` 的构造则是基于存储的旧的状态和 `reducer` 以及 `dispatch` 接收的 action 计算新的状态，然后触发组件的 re-render。

 ##### 3.4.3 useState

`useState` 的实现简单而巧妙：

```js
export function useState(initialState) {
	currentHook = 1;
	return useReducer(invokeOrReturn, initialState);
}
```

它基于 `useReducer` 实现，特殊之处在于 `reducer` 是 `invokeOrReturn` 函数。

这个函数作为 reducer 时，会判断接收到的 action 的类型。如果是函数类型，则调用 action 并传入上次的状态计算新的状态；如果是对象类型，则用接收到的 action 作为新的状态。

##### 3.4.4 useEffect

`useEffect` 本身做的事情不多，就是在指定参数发生变化的时候存储新的 Effect：

```js
export function useEffect(callback, args) {
	/** @type {import('./internal').EffectHookState} */
	const state = getHookState(currentIndex++, 3);
	if (!options._skipEffects && argsChanged(state._args, args)) {
		state._value = callback;
		state._args = args;

		currentComponent.__hooks._pendingEffects.push(state);
	}
}

// 比较方法
function argsChanged(oldArgs, newArgs) {
	return (
		!oldArgs ||
		oldArgs.length !== newArgs.length ||
		newArgs.some((arg, index) => arg !== oldArgs[index])
	);
}
```

`pendingEffects` 的使用在其他地方。

 `pendingEffects` 是在当前帧渲染完之后执行的回调，在 `options.diffed` 钩子中，会执行当前 VNode 对应的 Component 实例上的所有 `pendingEffects`。

为了确保是在当前帧渲染完毕后再执行回调，Preact 实现了以下方法：

```js
function afterNextFrame(callback) {
	const done = () => {
		clearTimeout(timeout);
		if (HAS_RAF) cancelAnimationFrame(raf);
		setTimeout(callback);
	};
	const timeout = setTimeout(done, RAF_TIMEOUT);

	let raf;
	if (HAS_RAF) {
		raf = requestAnimationFrame(done);
	}
}
```

这个方法会调用 `rAF` 函数在下一帧渲染之前注册一个 `setTimeout` 宏任务，所以这个宏任务的执行会在下一帧渲染完成之后。

为了确保这个方法一定会执行，Preact 还会注册一个定时任务（上方的 timeout）以确保 100ms 这个延时附近会执行回调。

执行回调的方法：

```js
function flushAfterPaintEffects() {
	afterPaintEffects.forEach(component => {
		if (component._parentDom) {
			try {
				component.__hooks._pendingEffects.forEach(invokeCleanup);
				component.__hooks._pendingEffects.forEach(invokeEffect);
				component.__hooks._pendingEffects = [];
			} catch (e) {
				component.__hooks._pendingEffects = [];
				options._catchError(e, component._vnode);
			}
		}
	});
	afterPaintEffects = [];
}
```

这个方法会依次执行上一次 render 得到的 `cleanup` 函数，然后依次执行当前 render 的 `pendingEffects`。

还有一种特殊情况，如果当前 render 还没来得及执行 `pendingEffects` 然后马上就开始了下一轮 render 怎么办？

Preact 通过在每次 render 之前清空所有 Component 实例上的所有 `pendingEffects` 来解决这个问题：

```js
options._render = vnode => {
	if (oldBeforeRender) oldBeforeRender(vnode);

	currentComponent = vnode._component;
	currentIndex = 0;

	const hooks = currentComponent.__hooks;
	if (hooks) {
		hooks._pendingEffects.forEach(invokeCleanup);
		hooks._pendingEffects.forEach(invokeEffect);
		hooks._pendingEffects = [];
	}
};
```

这样就不会漏掉来不及执行的 `pendingEffects` 了。

##### 3.4.5 useLayoutEffect

这个 Hook 的作用基本等同于类组件中的 `componentDidMount` 钩子，其实现和 `useEffect` 高度类似：

```js
export function useLayoutEffect(callback, args) {
	/** @type {import('./internal').EffectHookState} */
	const state = getHookState(currentIndex++, 4);
	if (!options._skipEffects && argsChanged(state._args, args)) {
		state._value = callback;
		state._args = args;

		currentComponent._renderCallbacks.push(state);
	}
}
```

区别在于这个 Hook 的回调存储在 Component 实例的 `_renderCallbacks` 属性中。

这个属性会在 `commit` 阶段开始之前执行，也就是在下一帧渲染之前。所以使用这个 Hook 做一些 DOM 样式变更可以防止用户观察到样式的变化过程。

##### 3.4.6 useMemo

useMemo 的实现也很简单：

```js
export function useMemo(factory, args) {
	/** @type {import('./internal').MemoHookState} */
	const state = getHookState(currentIndex++, 7);
	if (argsChanged(state._args, args)) {
		state._value = factory();
		state._args = args;
		state._factory = factory;
	}

	return state._value;
}
```

这个函数会在 args 参数变化的时候重新执行 factory 函数计算新的数据。

##### 3.4.7 useCallback

`useCallback` 基于 `useMemo` 实现：

```js
export function useCallback(callback, args) {
	currentHook = 8;
	return useMemo(() => callback, args);
}
```

很简单。

##### 3.4.8 useRef

`useRef` 同样基于 `useMemo` 实现：

```js
export function useRef(initialValue) {
	currentHook = 5;
	return useMemo(() => ({ current: initialValue }), []);
}
```

其原理在于维护了一个变量在 Component 实例中，每次 Hook 执行都可以通过引用传递获取到同一个内存中的数据，所以看起来 ref 是可以避开 Capture Value 的一种手段。

##### 3.4.9 useContext

`useContext` 的实现相对复杂。

先看 `createContext` 方法：

```js
export function createContext(defaultValue, contextId) {
	contextId = '__cC' + i++;

	const context = {
		_id: contextId,
		_defaultValue: defaultValue,
		/** @type {import('./internal').FunctionComponent} */
		Consumer(props, contextValue) {
			// return props.children(
			// 	context[contextId] ? context[contextId].props.value : defaultValue
			// );
			return props.children(contextValue);
		},
		/** @type {import('./internal').FunctionComponent} */
		Provider(props) {
			if (!this.getChildContext) {
				let subs = [];
				let ctx = {};
				ctx[contextId] = this;

				this.getChildContext = () => ctx;

				this.shouldComponentUpdate = function(_props) {
					if (this.props.value !== _props.value) {
						// I think the forced value propagation here was only needed when `options.debounceRendering` was being bypassed:
						// https://github.com/preactjs/preact/commit/4d339fb803bea09e9f198abf38ca1bf8ea4b7771#diff-54682ce380935a717e41b8bfc54737f6R358
						// In those cases though, even with the value corrected, we're double-rendering all nodes.
						// It might be better to just tell folks not to use force-sync mode.
						// Currently, using `useContext()` in a class component will overwrite its `this.context` value.
						// subs.some(c => {
						// 	c.context = _props.value;
						// 	enqueueRender(c);
						// });

						// subs.some(c => {
						// 	c.context[contextId] = _props.value;
						// 	enqueueRender(c);
						// });
						subs.some(enqueueRender);
					}
				};

				this.sub = c => {
					subs.push(c);
					let old = c.componentWillUnmount;
					c.componentWillUnmount = () => {
						subs.splice(subs.indexOf(c), 1);
						if (old) old.call(c);
					};
				};
			}

			return props.children;
		}
	};

	// Devtools needs access to the context object when it
	// encounters a Provider. This is necessary to support
	// setting `displayName` on the context object instead
	// of on the component itself. See:
	// https://reactjs.org/docs/context.html#contextdisplayname

	return (context.Provider._contextRef = context.Consumer.contextType = context);
}
```

`createContext` 返回一个对象，主要包含 `Consumer` 和 `Provider` 组件。

重点看 `Provider` 组件，当 `Provider` 组件更新的时候，shouldComponentUpdate 钩子会拿到所有的订阅者（都是 Component 实例）触发更新。

由于 shouldComponentUpdate 函数没有返回值，所以 Provider 组件并不会发生 re-render。

另外一个细节是 Provider 组件的实例会被挂载一个 `getChildContext` 函数，这个函数会在 diff 阶段执行并且把当前 context 合并到 `globalContext` 对象中。

再看 `useContext` 的实现：

```js
export function useContext(context) {
	const provider = currentComponent.context[context._id];
	// We could skip this call here, but than we'd not call
	// `options._hook`. We need to do that in order to make
	// the devtools aware of this hook.
	/** @type {import('./internal').ContextHookState} */
	const state = getHookState(currentIndex++, 9);
	// The devtools needs access to the context object to
	// be able to pull of the default value when no provider
	// is present in the tree.
	state._context = context;
	if (!provider) return context._defaultValue;
	// This is probably not safe to convert to "!"
	if (state._value == null) {
		state._value = true;
		provider.sub(currentComponent);
	}
	return provider.props.value;
}
```

这个 Hook 会拿到需要订阅的 Provider 组件的实例，然后在初次运行的时候订阅 Provider value 的变化。这样当 value 变化时，前文提到的 Provider 组件的 shouldComponentUpdate 钩子会触发所有的订阅者 re-render。



以上，是 Hooks 的具体实现。

### 四 解决问题

回头再看第二节提出的问题。

- **Hooks 的基本实现（包括数据结构）是怎么样的？**

  答：函数组件最终还是会基于 Preact.Component 进行实例化，Hooks 的状态会被关联到 Component 实例上。这样函数组件就可以拥有自己的状态了。
  不同的 Hooks 拥有不同的存储数据结构，具体参考 3.3 小节。

- **Preact 如何区分 renderCallbacks 和 pendingEffects？**

  答：`renderCallbacks` 在下一帧渲染前执行，会通过 `options._commit` 钩子触发回调的执行。

  而 `pendingEffects` 在下一帧渲染完成后执行，会通过 `rAF` 和 `setTimeout` 实现这一过程。

- **为什么 Hooks 要求每次 render 中 Hook 运行的顺序和数量必须一致？**

  答：因为 Hooks 根据一个索引在 Component 实例上存取关联的状态，而这个索引基于 Hooks 在当前组件内的执行顺序，如果两次 render 过程中 Hooks 的数量或者顺序发生了变化，那将会导致 Hooks 存取数据发生错误。

- **为什么有些 Hooks（useState…）可以触发 re-render？**

  答：Hooks 触发组件的 re-render 是通过调用组件的 `setState` 方法实现的（文章没有讲 `setState` 因为跟 Hooks 关系不大）。 

- **如何限制开发者使用 Hooks 的时机？**

  答：通过 Option Hooks 钩子限制只允许在 diff 阶段执行 Hooks，在非 diff 阶段执行 Hooks 会导致相关错误抛出。

  具体参考 3.4.1 小节。