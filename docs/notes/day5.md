# Tapable 架构 02

今天这一节继续讲解 `SyncHook`，来看一下钩子内部是怎么实现的。

还是从创建钩子的角度来看：

```javascript
// SyncHook.js
const Hook = require('./Hook');
const HookCodeFactory = require('./HookCodeFactory');

class SyncHookCodeFactory extends HookCodeFactory {
  content({ onError, onResult, onDone, rethrowIfPossible }) {
    return this.callTapsSeries({
      onError: (i, err) => onError(err),
      onDone,
      rethrowIfPossible,
    });
  }
}

const factory = new SyncHookCodeFactory();

class SyncHook extends Hook {
  tapAsync() {
    throw new Error('tapAsync is not supported on a SyncHook');
  }

  tapPromise() {
    throw new Error('tapPromise is not supported on a SyncHook');
  }

  compile(options) {
    factory.setup(this, options);
    return factory.create(options);
  }
}

module.exports = SyncHook;
```

先忽略 `SyncHookCodeFactory` 及其父类 `HookCodeFactory`。对于 `SyncHook` 来说，是 `Hook` 的派生类，并且重写了 `tapAsync()` 和 `tapPromise()`（禁止调用这两个 API ），从名字上可以看出是这是两个挂载异步钩子的函数，所以也忽略。而 `compile()` 方法也看不出来什么东西，没有看到 `tap()` 和 `call()`，所以需要再去看下它的基类 `Hook` 的源码。

```javascript
// Hook.js
class Hook {
  constructor(args) {
    // args 就是钩子回调函数的形参
    if (!Array.isArray(args)) args = [];
    // 备份
    this._args = args;
    // taps 就是所有挂载在钩子上的插件
    this.taps = [];
    // interceptors 拦截器，可以用来对 call，tap 等方法做拦截
    this.interceptors = [];
    // 出现了 call，而 _createCompileDelegate 就是用来编译钩子的代理函数，并且内部通过懒加载的方式，即直到调用 call 时才真正的编译钩子，而编译其实就是 SyncHook 中重写的 compile 方法
    // 也保存备份，_call（未编译）
    this.call = this._call = this._createCompileDelegate('call', 'sync');
    // 与 call 的功能一样，但是是异步的，先忽略
    this.promise = this._promise = this._createCompileDelegate(
      'promise',
      'promise'
    );
    this.callAsync = this._callAsync = this._createCompileDelegate(
      'callAsync',
      'async'
    );
    // 编程中最难的两件事之一：给变量起名 🙂
    this._x = undefined;
  }

  compile(options) {
    // 派生类必须实现自己的编译函数
    throw new Error('Abstract: should be overriden');
  }

  _createCall(type) {
    return this.compile({
      taps: this.taps,
      interceptors: this.interceptors,
      args: this._args,
      type: type,
    });
  }

  _createCompileDelegate(name, type) {
    const lazyCompileHook = (...args) => {
      // 重写 call 函数，从上面的 _createCall 可以看出，call 函数其实就是派生类编译 compile 的结果。在执行编译时，会传入这个钩子上挂载的插件（tags），拦截器（interceptors），形参（args）以及类型（type）。
      this[name] = this._createCall(type);
      // 调用编译后的 call 函数，并传递实参
      return this[name](...args);
    };
    return lazyCompileHook;
  }

  tap(options, fn) {
    // options 是插件（名），fn 是回调函数
    if (typeof options === 'string') options = { name: options };
    if (typeof options !== 'object' || options === null)
      throw new Error(
        'Invalid arguments to tap(options: Object, fn: function)'
      );
    // 格式化 options，默认类型是同步的
    options = Object.assign({ type: 'sync', fn: fn }, options);
    if (typeof options.name !== 'string' || options.name === '')
      throw new Error('Missing name for tap');
    // 添加拦截器
    options = this._runRegisterInterceptors(options);
    // 挂载插件
    this._insert(options);
  }

  tapAsync(options, fn) {
    // 跟 tap 类似，不过是异步的，先忽略
  }

  tapPromise(options, fn) {
    // 跟 tap 类似，不过是异步的，先忽略
  }

  _runRegisterInterceptors(options) {
    for (const interceptor of this.interceptors) {
      if (interceptor.register) {
        const newOptions = interceptor.register(options);
        if (newOptions !== undefined) options = newOptions;
      }
    }
    return options;
  }

  withOptions(options) {
    // 暂时用不到，忽略
  }

  isUsed() {
    return this.taps.length > 0 || this.interceptors.length > 0;
  }

  intercept(interceptor) {
    this._resetCompilation();
    this.interceptors.push(Object.assign({}, interceptor));
    if (interceptor.register) {
      for (let i = 0; i < this.taps.length; i++)
        this.taps[i] = interceptor.register(this.taps[i]);
    }
  }

  _resetCompilation() {
    // 重置 call 方法至未编译状态
    this.call = this._call;
    this.callAsync = this._callAsync;
    this.promise = this._promise;
  }

  _insert(item) {
    // item 就是插件（上面传入的变量叫 options，但是感觉就像是插件）
    // 每次挂载新的插件时，都会重置钩子的编译状态
    this._resetCompilation();
    // 插件中 before 字段，用来插队。
    let before;
    if (typeof item.before === 'string') before = new Set([item.before]);
    else if (Array.isArray(item.before)) {
      before = new Set(item.before);
    }
    // 默认 stage（可以理解为优先级）
    let stage = 0;
    if (typeof item.stage === 'number') stage = item.stage;

    // 准备工作已经结束，准备插入
    // this.taps 数组就是钩子上挂载的所有插件；默认情况下新挂载的插件会在数组最右侧插入
    // 像这样 [first, second, third...]
    let i = this.taps.length;
    while (i > 0) {
      i--;
      // 如果不设置 before 或 stage, 则仅仅复制最后一个插件到右侧，并改写它。
      // 如果设置有 before 或 stage, 则从右到左，将每个插件右移一位，直至没有 before 或者 stage 不小于当前插件
      const x = this.taps[i];
      this.taps[i + 1] = x;
      const xStage = x.stage || 0;
      if (before) {
        if (before.has(x.name)) {
          before.delete(x.name);
          continue;
        }
        if (before.size > 0) {
          continue;
        }
      }
      if (xStage > stage) {
        continue;
      }
      i++;
      break;
    }
    this.taps[i] = item;
  }
}

module.exports = Hook;
```

唯一一个比较难的方法就是挂载插件时调用的 `_insert()` 方法，这里面实现了两种特性：`before` 和 `stage`。可以用来定义插件的顺序（虽然 Webpack 中没有用到这些，但是还是讲一下吧）。举一个简单的例子：

```javascript
const hook = new SyncHook();

hook.tap('A', () => console.log('This is A.'));
hook.tap('B', () => console.log('This is B.'));
hook.tap('C', () => console.log('This is C.'));
```

我们对一个钩子挂载了 3 个无聊的插件，也就是说会分别执行 `_insert()` 3 次，我们来看一下每次 while 循环时的 `taps` 属性的变化：

- `_insert({name: A, ...})`: 并不会执行 while 循环，所以 `taps = [{name: A, ...}]`;
- `_insert({name: B, ...})`:
  - `i = 1`，`taps = [{name: A, ...}, {name: A, ...}]`;
  - `break` 跳出循环，`taps = [{name: A, ...}, {name: B, ...}]`;
- `_insert({name: C, ...})`:
  - `i = 2`，`taps = [{name: A, ...}, {name: B, ...}, {name: B, ...}]`;
  - `break` 跳出循环，`taps = [{name: A, ...}, {name: B, ...}, {name: C, ...}]`;

此时我们再添加一个插件：

```javascript
hook.tap({ name: 'D', before: 'B' }, () => console.log('This is D.'));
```

继续上面的步骤的话：

- `_insert({name: D, ...})`:
  - `i = 3`，`taps = [{name: A, ...}, {name: B, ...}, {name: C, ...}, {name: C, ...}]`，存在 `before`，但 `before !== 'C'`，继续循环
  - `i = 2`，`taps = [{name: A, ...}, {name: B, ...}, {name: B, ...}, {name: C, ...}]`，存在 `before`，且 `before === 'B'`，删除后继续循环
  - `i = 1`，`taps = [{name: A, ...}, {name: A, ...}, {name: B, ...}, {name: C, ...}]`，存在 `before`，但大小为空
  - `break` 跳出循环，`taps = [{name: A, ...}, {name: D, ...}, {name: B, ...}, {name: C, ...}]`;

这看起来很绕（为了减小时间复杂度），但其实 `taps` 属性就是一个带有优先级（`stage`）的队列，左侧是头部，右侧是尾部，并且可以使用可选的 `before` 来插队（用来直接指定在哪些插件的前面插入插件）。

插入的过程就是从右到左遍历所有已挂载的插件，查看该插件是否满足插队条件：

- 不满足就直接插在队尾；
- 满足条件，就需要继续遍历，直至找到满足条件的插件，然后插在其前面（左面）；

# 总结

放一下官网 README 中涉及的接口：

```javascript

// 共有方法
interface Hook {
  tap: (name: string | Tap, fn: (context?, ...args) => Result) => void,
}

interface Tap {
  name: string,
  type: string
  fn: Function,
  stage: number,
}

// 私有方法 (只有包含钩子的类才可以调用):
interface Hook {
  call: (...args) => Result,
}
```

本节主要介绍了同步钩子 `SyncHook` 实例的属性（`taps`, `args`），以及使用 `Hook.prototype.tap()` 到底如果挂载插件的，包括 `before` 和 `stage` 两个属性如何设置插件顺序。明天再讲如何来编译钩子并且如何执行其回调函数。