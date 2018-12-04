# Tapable 架构 04

对于 SyncHook，它重写了 `tapAsync` 和 `tapPromise` 两个方法来禁止挂载异步类型的插件，但通过之前的解读可以发现，在编译钩子的执行函数时，依然编译了异步执行方法 `callAsync` 和 `promise`。也就是说，我们可以在同步钩子上使用异步执行方法。

在了解为什么这么做之前，先来看看两种异步执行函数的代码：

```javascript
// callAsync.toString()
function anonymous(_callback) {
  "use strict";
  var _context;
  var _x = this._x;
  var _fn0 = _x[0];
  var _hasError0 = false;
  try {
    _fn0();
  } catch (_err) {
    _hasError0 = true;
    _callback(_err);
  }
  if (!_hasError0) {
    var _fn1 = _x[1];
    var _hasError1 = false;
    try {
      _fn1();
    } catch (_err) {
      _hasError1 = true;
      _callback(_err);
    }
    if (!_hasError1) {
      var _fn2 = _x[2];
      var _hasError2 = false;
      try {
        _fn2();
      } catch (_err) {
        _hasError2 = true;
        _callback(_err);
      }
      if (!_hasError2) {
        _callback();
      }
    }
  }
}

// promise.toString()
function anonymous() {
  "use strict";
  return new Promise((_resolve, _reject) => {
    var _sync = true;
    var _context;
    var _x = this._x;
    var _fn0 = _x[0];
    var _hasError0 = false;
    try {
      _fn0();
    } catch (_err) {
      _hasError0 = true;
      if (_sync)
        _resolve(
          Promise.resolve().then(() => {
            throw _err;
          })
        );
      else _reject(_err);
    }
    if (!_hasError0) {
      var _fn1 = _x[1];
      var _hasError1 = false;
      try {
        _fn1();
      } catch (_err) {
        _hasError1 = true;
        if (_sync)
          _resolve(
            Promise.resolve().then(() => {
              throw _err;
            })
          );
        else _reject(_err);
      }
      if (!_hasError1) {
        var _fn2 = _x[2];
        var _hasError2 = false;
        try {
          _fn2();
        } catch (_err) {
          _hasError2 = true;
          if (_sync)
            _resolve(
              Promise.resolve().then(() => {
                throw _err;
              })
            );
          else _reject(_err);
        }
        if (!_hasError2) {
          _resolve();
        }
      }
    }
    _sync = false;
  });
}
```

其实两种异步执行方式的插件执行方式是一样的，不同的只是插件报错时的调用方式。`callAsync` 通过 NodeJS 中常见的将错误作为第一个参数的方式；而 `promise` 则是 resolve 并传递错误的 Promise。

这两种执行方式对于插件的顺序执行大抵过程为：

1. 执行插件回调，如果出错，就执行回调并传递错误；
2. 如果没有出错，就返回 1 执行下一个插件回调；
3. 知道所有的插件执行完毕，执行回调；

相比于同步执行方法，只是多了执行的回调函数而已，即当全部插件回调都执行完毕或者某个插件回调函数报错时调用的函数。

在实际开发中，通常应用于钩子的携带者，监听所有插件执行完毕的事件，比如说 webpack 在一个周期上挂载的所有插件执行完毕之后，继续执行编译。

另外，对于异步钩子的 `tapAsync` 和 `tapPromise` 方法，其内部实现与 `tap` 方法 90% 都是重复的。唯一不同之处在于插件配置中的 `type` 字段以及调用出错时的错误信息。而这个 `type` 字段用于在编译执行函数时进行 `switch`。

# 总结

本节对同步钩子剩余的一点内容：异步执行方式，进行了并不深入的解读，了解了同步钩子的异步执行函数的工作原理。另外还了解了下异步挂载方式与同步挂载方式没什么两样。

通过这两节的内容，相信你也察觉出其中的一点问题：对于超过 50% 以上内容不同的编译函数来说，这里通过传参的方式实现，所谓的派生类执行函数工厂，“仅仅”在重写 `content` 时传递不同的参数。导致即便使用了派生类区分不同类型的钩子，但在编译执行函数的过程中 99% 的逻辑依然在基类中，阅读起来十分痛苦。
