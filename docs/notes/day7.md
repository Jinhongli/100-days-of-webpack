# Tapable 架构 04

上一节讲过了 SyncHook 的执行方法 `call()`，这是一个同步的执行方法，此外还有另外两个执行函数：`callAsync()` 和 `promise` 。显然，这是异步的执行方法。

🤨 这不是 SyncHook 同步钩子么，为什么会有异步执行方式？首先来看下编译出来的异步执行方法长什么样子（假设挂载 3 个插件）：

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
  return new Promise((_resolve,_reject)=>{
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
        _resolve(Promise.resolve().then(()=>{
          throw _err;
        }
        ));
      else
        _reject(_err);
    }
    if (!_hasError0) {
      var _fn1 = _x[1];
      var _hasError1 = false;
      try {
        _fn1();
      } catch (_err) {
        _hasError1 = true;
        if (_sync)
          _resolve(Promise.resolve().then(()=>{
            throw _err;
          }
          ));
        else
          _reject(_err);
      }
      if (!_hasError1) {
        var _fn2 = _x[2];
        var _hasError2 = false;
        try {
          _fn2();
        } catch (_err) {
          _hasError2 = true;
          if (_sync)
            _resolve(Promise.resolve().then(()=>{
              throw _err;
            }
            ));
          else
            _reject(_err);
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

可以看出，两个执行函数在调用时，挂载的回调函数的执行依然是同步的，只不过会添加 `try/catch` 块用于回调函数出错后执行 `_callback` 或者修改 Promise 状态。

这么看来，`callAsync()` 和 `promise()` 仅仅是增加了所有插件回调函数执行完毕之后的回调函数。如果其中一个插件报错，那么就会将错误作为第一个参数调用回调。可以说，使用异步执行方式可以对所有插件的回调函数的执行添加一个回调函数。不同的是，`callAsync()` 的回调函数的执行也是同步的，而 `promise()` 的回调函数是异步的（等待线程为空之后才会调用）。

而对于插件的回调函数来说，其调用顺序与 `call()` 来比，并没有什么不同。

而具体 3 个执行方法的拼接，是 `HookCodeFactory.prototype.create()` 以及 `HookCodeFactory.prototype.callTap()` 两个方法根据 `type` 类型拼接不同 JavaScript 语句得出的。

## 总结

虽然 SyncHook 是同步钩子，禁止了 `tapAsync()` 和 `tapPromise()` 的异步挂载插件的方法，但是执行方法依然可以有异步的：`callAsync()` 和 `promise()`。

相比 `call()` 只是多了所有插件执行完毕的回调函数（符合 Nodejs 回调函数规范，即如果有插件报错，也会执行该回调，会讲错误传递作为第一个参数传递给回调函数），而所有插件的执行依然是同步的，并且 `callAsync()` 的回调函数也是同步的，只有 `promise()` 才是异步的。