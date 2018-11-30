# Tapable 架构 02

本节依然以最简单的 SyncHook 为例，稍微深入的讲一下 Hook 是如何生成的以及执行函数是如何编译的，至于插件挂载的细节暂且略过。

废话少说，先看源码：

```js
// SyncHook.js
const Hook = require("./Hook");
const HookCodeFactory = require("./HookCodeFactory");

class SyncHookCodeFactory extends HookCodeFactory {
  content({ onError, onResult, onDone, rethrowIfPossible }) {
    return this.callTapsSeries({
      onError: (i, err) => onError(err),
      onDone,
      rethrowIfPossible
    });
  }
}

const factory = new SyncHookCodeFactory();

class SyncHook extends Hook {
  tapAsync() {
    throw new Error("tapAsync is not supported on a SyncHook");
  }

  tapPromise() {
    throw new Error("tapPromise is not supported on a SyncHook");
  }

  compile(options) {
    factory.setup(this, options);
    return factory.create(options);
  }
}

module.exports = SyncHook;
```

首先就是 SyncHook 继承自 Hook，并且重写了异步挂载方法 `tapAsync()` 和 `tapPromise()`，以及编译函数 `compile()`。

值得注意是 `compile` 函数中出现了 `SyncHookCodeFactory` 的一个实例 `factory`，从命名可以猜出这个类用于生成执行函数，而且继承自基类 `HookCodeFactory`。但依然看不出来钩子是如何创建的，所以继续看 Hook 的源码。

```js
// Hook.js
class Hook {
  constructor(args) {
    if (!Array.isArray(args)) args = [];
    // 插件回调函数所需的形参
    this._args = args;
    // 所有挂载在钩子上的插件
    this.taps = [];
    // 拦截器，可以用来对 call，tap 等方法做拦截，高级应用先忽略
    this.interceptors = [];
    // 执行函数，通过懒执行的方式，实现真正运行 call 时才进行编译
    // 同时也保存一个备份，_call（未编译状态），用于重置执行函数
    this.call = this._call = this._createCompileDelegate("call", "sync");
    // 与 call 的功能一样，但是是异步的，先忽略
    this.promise = this._promise = this._createCompileDelegate(
      "promise",
      "promise"
    );
    this.callAsync = this._callAsync = this._createCompileDelegate(
      "callAsync",
      "async"
    );
    // 不知道为什么要取这么个诡秘的名字，之后就能知道它是干嘛滴
    this._x = undefined;
  }

  compile(options) {
    // 派生类必须重写编译函数
    throw new Error("Abstract: should be overriden");
  }

  _createCall(type) {
    // 真正执行编译的函数
    return this.compile({
      taps: this.taps,
      interceptors: this.interceptors,
      args: this._args,
      type: type
    });
  }

  _createCompileDelegate(name, type) {
    const lazyCompileHook = (...args) => {
      // 运行编译函数得到执行函数（调用派生类 compile 方法的返回值）。
      // 在执行编译时，会传入这个钩子上挂载的插件（tags），拦截器（interceptors），形参（args）以及类型（type）。
      this[name] = this._createCall(type);
      // 调用执行函数，且透传执行 call 时的实参
      return this[name](...args);
    };
    return lazyCompileHook;
  }
}

module.exports = Hook;
```

可以看出在实例化钩子时声明了一些上一节露过脸的属性和方法。其中钩子的执行函数就是 `call`，`promise`，`callAsync` 三个。并且可以看出，执行函数并不是在实例化的时候编译的，而是先创建编译的代理，在真正调用执行函数时，先运行派生类的 `compile` 方法进行编译得到执行函数，然后再运行这个执行函数并返回结果。

在之前的代码已经知道，`SyncHook.prototype.compile` 方法是通过执行函数工厂 `SyncHookCodeFactory` 的一个实例运行 `setup` 之后，运行 `create` 之后的返回值。从名字上可以猜出来：先配置，后创建。

在继续看具体编译过程之前，先来看一下编译后的执行函数到底长什么样子，一个简陋的例子:

```js

const hook = new SyncHook();

hook.tap("A", () => console.log("This is A."));
hook.tap("B", () => console.log("This is B."));
hook.tap("C", () => console.log("This is C."));
hook.call();

// console.log(hook.call.toString())
function anonymous() {
  "use strict";
  var _context;
  var _x = this._x;
  var _fn0 = _x[0];
  _fn0();
  var _fn1 = _x[1];
  _fn1();
  var _fn2 = _x[2];
  _fn2();
}
```

上面的代码就是在编译执行之后，打印出来的执行函数。整个函数的意图比较简单，就是顺序执行所有插件的回调函数。但是出现了之前不知道是什么玩意的 `_x`，大概能猜出来它是用来存放所有插件的回调函数的属性。至于为什么执行所有插件回调函数要用这么诡异的方式，而不是直接遍历 `taps` 执行回调，在后面学习更多类型的钩子就知道了。

现在终于可以看编译执行函数的具体过程了。因为代码实在太长，分别截取关键方法来看，首先是 `setup`。

```js
class Hook {
  _createCall(type) {
		return this.compile({
			taps: this.taps,
			interceptors: this.interceptors,
			args: this._args,
			type: type
		});
	}
}

class SyncHook extends Hook
	compile(options) {
		factory.setup(this, options);
		return factory.create(options);
  }
}

class HookCodeFactory {
  setup(instance, options) {
		instance._x = options.taps.map(t => t.fn);
	}
}
```

`setup` 这个方法的作用非常简单，就是遍历所有的插件，将所有插件的回调函数放在钩子的实例属性 `_x` 上。另外值得注意的是，在调用 `compile` 时传递的实参就是钩子的配置，包含 `taps`，`interceptors`，`args`，`type` 四项。

然后是核心的 `create` 方法，用来真正创建执行函数。

```js
class SyncHookCodeFactory extends HookCodeFactory {
	content({ onError, onResult, onDone, rethrowIfPossible }) {
    // 重写 content 方法
		return this.callTapsSeries({
			onError: (i, err) => onError(err),
			onDone,
			rethrowIfPossible
		});
	}
}

class HookCodeFactory {
  constructor(config) {
		this.config = config;
		this.options = undefined;
		this._args = undefined;
	}

	create(options) {
    // 执行函数工厂初始化，传递钩子配置和执行函数形参
		this.init(options);
    let fn;
    // 这里先省略异步执行函数的分支
		switch (this.options.type) {
      case "sync":
        // 由 Function 创建一个新的函数，其形参是钩子配置的 args 字段的复制
        // 而函数体则是由 header() 和 content() 执行之后拼接在一起的内容
        // 并分别设置函数体
				fn = new Function(
					this.args(),
					'"use strict";\n' +
						this.header() +
						this.content({
							onError: err => `throw ${err};\n`,
							onResult: result => `return ${result};\n`,
							onDone: () => "",
							rethrowIfPossible: true
						})
				);
				break;
			case "async":
				break;
			case "promise":
				break;
    }
    // 重置执行函数工厂
    this.deinit();
    // 返回执行函数
		return fn;
  }
  
	/**
	 * @param {{ type: "sync" | "promise" | "async", taps: Array<Tap>, interceptors: Array<Interceptor> }} options
	 */
	init(options) {
		this.options = options;
		this._args = options.args.slice();
	}

	deinit() {
		this.options = undefined;
		this._args = undefined;
  }
  
  header() {
    // 得到 执行函数体 头部
  }
  
  needContext() {
    // 判断是否由钩子需要设定执行环境
		for (const tap of this.options.taps) if (tap.context) return true;
		return false;
  }

  callTap(tapIndex, { onError, onResult, onDone, rethrowIfPossible }) {
    // 获取一个插件回调函数的执行模板
	}
  
  callTapsSeries({ onError, onResult, onDone, rethrowIfPossible }) {
    // 得到 顺序执行所有插件回调函数 的函数体
		const next = i => {
      // 得到当前执行函数的模板, i 表示插件的索引
		};
		// 从第一个插件开始
		return next(0);
  }
  
  args({ before, after } = {}) {
    // 对执行函数的形参进行格式化，可以通过 before 和 after 在参数列表首尾添加额外参数
    // 在 new Function 生成执行函数处，以及生成插件回调运行时传递
	}
}
```

这里先不考虑生成具体函数体的构成细节，只考虑大体上的编译流程：

1. 执行 `compile()` 会先运行执行函数工厂的 `setup()`，目的是保存所有插件回调函数；
2. 然后执行执行函数工厂的 `create()`，内部首先调用 `init()` 执行初始化，然后根据钩子的类型使用 `new Function()` 生成执行函数，最后重置工厂参数后返回执行函数；

使用 `new Function` 生成执行函数的细节：

1. 执行函数的参数由 `args()` 进行格式化；
2. 执行函数的函数体由 `header()` 和 `content()` 拼接而成；
3. `header()` 负责执行函数头部内容，包括初始化插件回调的执行环境、调用执行函数拦截器
4. `content()` 负责将所有插件回调函数的执行凭借在一起。对于不同类型的钩子，通过重写 `content` 方法、传入不同的参数来生成不同的执行代码。

`content()` 接受的参数包括：

- `onError()`，处理插件回调执行报错的代码
- `onResult()`，插件回调返回值相关的代码
- `onDone()`，执行完毕时的代码
- `rethrowIfPossible`：对每个插件回调添加容错的代码；（插件回调报错后是否继续执行剩余插件）

执行 `content()` 的内部流程大致是：

1. 根据钩子类型执行 `callTapsSeries`，`callTapsLooping` 或 `callTapsParallel`，以不同方式遍历插件回调；
2. 不论哪种遍历方式，都使用 `callTap` 来插入一个插件回调的执行代码；

# 总结

本节粗略的介绍了钩子实例的初始化以及编译执行函数的过程。对于不同种类的钩子，以及不同类型的插件挂载方式，整体流程都是一样的，唯一不同的就是 `content` 函数生成的所有插件回调的执行代码。

至于对不同类型钩子编译执行函数的更多细节以后再细说。
