# Tapable 架构 03

上一节讲了使用 `Hook.prototype.tap()` 内部是如何挂载插件的，今天来讲另外一个重要的方法：`hook.call()`。

在基类 `Hook` 的构造函数中，钩子实例的 `call()` 方法是 `_createCompileDelegate()` 的返回值。

```javascript
// Hook.js
class Hook {
  constructor(args) {
    // ...
    // 出现了 call，而 _createCompileDelegate 就是用来编译钩子的代理函数，并且内部通过懒加载的方式，即直到调用 call 时才真正的编译钩子，而编译其实就是 SyncHook 中重写的 compile 方法
    // 也保存备份，_call（未编译）
    this.call = this._call = this._createCompileDelegate('call', 'sync');
    // ...
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
}
```

```javascript
// SyncHook.js
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
  compile(options) {
    factory.setup(this, options);
    return factory.create(options);
  }
}
```

从上面两个文件可以参数，`call()` 方法会在调用时才进行编译，编译的结果是由 `SyncHookCodeFactory` 的实例 `factory.create()` 的结果，从变量的命名也可以看出，`SyncHookCodeFactory` 是用来创建函数的工厂函数，继承自 `HookCodeFactory` 并重写了其中的 `content()` 方法。

所以我们继续看 `HookCodeFactory` 这个类的内部，了解到底是如何编译的（省略同步钩子用不到的方法）。

```javascript
// HookCodeFactory.js
class HookCodeFactory {
  constructor(config) {
    this.config = config;
    this.options = undefined;
  }

  create(options) {
    this.init(options);
    switch (this.options.type) {
      case 'sync':
        // 创建 sync 类型的执行函数
        return new Function(
          this.args(),
          '"use strict";\n' +
            this.header() +
            this.content({
              onError: err => `throw ${err};\n`,
              onResult: result => `return ${result};\n`,
              onDone: () => '',
              rethrowIfPossible: true,
            })
        );
      case 'async':
        // 创建 async 类型的执行函数
      case 'promise':
        // 创建 promise 类型的执行函数
    }
  }

  setup(instance, options) {
    instance._x = options.taps.map(t => t.fn);
  }

  /**
   * @param {{ type: "sync" | "promise" | "async", taps: Array<Tap>, interceptors: Array<Interceptor> }} options
   */
  init(options) {
    this.options = options;
    this._args = options.args.slice();
  }

  header() {
    let code = '';
    if (this.needContext()) {
      code += 'var _context = {};\n';
    } else {
      code += 'var _context;\n';
    }
    code += 'var _x = this._x;\n';
    if (this.options.interceptors.length > 0) {
      code += 'var _taps = this.taps;\n';
      code += 'var _interceptors = this.interceptors;\n';
    }
    for (let i = 0; i < this.options.interceptors.length; i++) {
      const interceptor = this.options.interceptors[i];
      if (interceptor.call) {
        code += `${this.getInterceptor(i)}.call(${this.args({
          before: interceptor.context ? '_context' : undefined,
        })});\n`;
      }
    }
    return code;
  }

  needContext() {
    for (const tap of this.options.taps) if (tap.context) return true;
    return false;
  }

  callTap(tapIndex, { onError, onResult, onDone, rethrowIfPossible }) {
    let code = '';
    let hasTapCached = false;
    for (let i = 0; i < this.options.interceptors.length; i++) {
      const interceptor = this.options.interceptors[i];
      if (interceptor.tap) {
        if (!hasTapCached) {
          code += `var _tap${tapIndex} = ${this.getTap(tapIndex)};\n`;
          hasTapCached = true;
        }
        code += `${this.getInterceptor(i)}.tap(${
          interceptor.context ? '_context, ' : ''
        }_tap${tapIndex});\n`;
      }
    }
    code += `var _fn${tapIndex} = ${this.getTapFn(tapIndex)};\n`;
    const tap = this.options.taps[tapIndex];
    switch (tap.type) {
      case 'sync':
        if (!rethrowIfPossible) {
          code += `var _hasError${tapIndex} = false;\n`;
          code += 'try {\n';
        }
        if (onResult) {
          code += `var _result${tapIndex} = _fn${tapIndex}(${this.args({
            before: tap.context ? '_context' : undefined,
          })});\n`;
        } else {
          code += `_fn${tapIndex}(${this.args({
            before: tap.context ? '_context' : undefined,
          })});\n`;
        }
        if (!rethrowIfPossible) {
          code += '} catch(_err) {\n';
          code += `_hasError${tapIndex} = true;\n`;
          code += onError('_err');
          code += '}\n';
          code += `if(!_hasError${tapIndex}) {\n`;
        }
        if (onResult) {
          code += onResult(`_result${tapIndex}`);
        }
        if (onDone) {
          code += onDone();
        }
        if (!rethrowIfPossible) {
          code += '}\n';
        }
        break;
      case 'async':
        // ...
      case 'promise':
        // ...
    }
    return code;
  }

  callTapsSeries({ onError, onResult, onDone, rethrowIfPossible }) {
    if (this.options.taps.length === 0) return onDone();
    const firstAsync = this.options.taps.findIndex(t => t.type !== 'sync');
    const next = i => {
      if (i >= this.options.taps.length) {
        return onDone();
      }
      const done = () => next(i + 1);
      const doneBreak = skipDone => {
        if (skipDone) return '';
        return onDone();
      };
      return this.callTap(i, {
        onError: error => onError(i, error, done, doneBreak),
        onResult:
          onResult &&
          (result => {
            return onResult(i, result, done, doneBreak);
          }),
        onDone:
          !onResult &&
          (() => {
            return done();
          }),
        rethrowIfPossible:
          rethrowIfPossible && (firstAsync < 0 || i < firstAsync),
      });
    };
    return next(0);
  }

  args({ before, after } = {}) {
    let allArgs = this._args;
    if (before) allArgs = [before].concat(allArgs);
    if (after) allArgs = allArgs.concat(after);
    if (allArgs.length === 0) {
      return '';
    } else {
      return allArgs.join(', ');
    }
  }

  getTapFn(idx) {
    return `_x[${idx}]`;
  }

  getTap(idx) {
    return `_taps[${idx}]`;
  }

  getInterceptor(idx) {
    return `_interceptors[${idx}]`;
  }
}
```

`HookCodeFactory` 创建实例时并没有什么东西，所以就不讲了。继续讲 `SyncHook.prototype.compile()` 方法。

首先会执行 `factory.setup()` 方法，在这个函数中终于看到了在 `Hook` 构造函数中出现的 `_x` 属性，其实就是**所有插件回调函数的数组**。

然后在执行 `factory.create()` 方法，这个方法就是用来创建执行函数的，生成的执行函数也就是编译结果。其内部通过 `new Function()` 来创建执行函数，执行函数的形参是插件中设置的参数（执行 `compile()` 时传入的 `args`），内容则是由 `header()` 和 `content()` 拼接出来的，一个最简单的编译好的执行函数就像这样：

```javascript
// 假设挂载了 3 个插件
function (arg1, arg2){
  "use strict";
  // this.header() 的结果
  var _context;
  var _x = this._x;
  // next(0) 的结果
  var _fn0 = _x[0];
  _fn0();
  // next(1) 的结果
  var _fn1 = _x[1];
  _fn1();
  // next(2) 的结果
  var _fn2 = _x[2];
  _fn2();
}
```

内部非常简单，按照顺序挨个执行插件的回调函数，这个执行函数就会被赋给 `hook.call`。

另外一点就是，`call()` 并不是公共方法，而是只能给包含钩子的类使用。

🤔 这什么意思？假设我们有两个开发者共同参与开发一个 Tapable 架构的库：A 来开发核心功能模块，B 来开发扩展插件。这个库的钩子 A 来定义并且使用 `call()` 来调用，而对于 B 来说只需要知道钩子执行的时机，并使用 `tap()` 将自己的插件挂载在对应的钩子上就可以了。

## 总结

总的来说，钩子实例上的 `hook.call` 方法用来执行挂载在钩子上的回调函数的方法，该方法在首次调用时才会真正的编译（编译的过程就是遍历所有的插件，然后使用 `new Function` 拼接出一个执行函数），并将编译后的执行函数重新赋值给 `hook.call`（此时会保留其编译前的状态 `hook._call` 以便新增插件后重新编译）。之后再调用 `hook.call` 就是直接运行执行函数。

至此，SyncHook 的主要功能（使用 `tap()` 挂载插件，使用 `call()` 执行回调）就介绍完了。