# Tapable 架构 03

目前，我们已经知道了如何声明和使用钩子，钩子的执行函数的编译流程，本节会对插件挂载的具体细节进行解读。

仍然是先上源码：

```js
class Hook {
  tap(options, fn) {
    if (typeof options === "string") options = { name: options };
    if (typeof options !== "object" || options === null)
      throw new Error(
        "Invalid arguments to tap(options: Object, fn: function)"
      );
    options = Object.assign({ type: "sync", fn: fn }, options);
    if (typeof options.name !== "string" || options.name === "")
      throw new Error("Missing name for tap");
    options = this._runRegisterInterceptors(options);
    this._insert(options);
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

  _insert(item) {
    this._resetCompilation();
    let before;
    if (typeof item.before === "string") before = new Set([item.before]);
    else if (Array.isArray(item.before)) {
      before = new Set(item.before);
    }
    let stage = 0;
    if (typeof item.stage === "number") stage = item.stage;
    let i = this.taps.length;
    while (i > 0) {
      i--;
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

  _resetCompilation() {
    this.call = this._call;
    this.callAsync = this._callAsync;
    this.promise = this._promise;
  }
}
```

这里只展示了 `tap` 这一个挂载方式。其内部先对插件格式化，然后在插件挂载之前，运行拦截器（比如说想要对每个插件执行统一操作，增加个属性之类的），最后通过 `_insert()` 将插件添加至钩子实例的 `taps` 列表中。

挂载插件时支持 `before` 的，但是我们先忽略它，看看插件是如何插入 `taps` 中的。我们将 `_insert()` 简化为：

```js
_insert(item) {
  this._resetCompilation();
  let stage = 0;
  if (typeof item.stage === "number") stage = item.stage;
  let i = this.taps.length;
  while (i > 0) {
    i--;
    const x = this.taps[i];
    this.taps[i + 1] = x;
    const xStage = x.stage || 0;
    if (xStage > stage) {
      continue;
    }
    i++;
    break;
  }
  this.taps[i] = item;
}
```

首先会执行 `_resetCompilation()` 来重置执行函数的编译结果（看了上一节之后就应该知道为什么了），重置的方式也比较简单，直接赋值为初始状态。

然后会初始化插件的 `state`，可以理解为插件的优先级，默认为 `0`，`stage` 越大优先级越高，越靠近列表头部。

接下来就是遍历已有插件，并在合适位置插入了。该遍历是倒序的，即从最后一项开始遍历。一旦 `stage` 小于等于当前项，就停止遍历，将插件放置其后。

如果考虑 `before`，插件插入的位置就会更加灵活：

```js
_insert(item) {
  this._resetCompilation();
  let before;
  if (typeof item.before === "string") before = new Set([item.before]);
  else if (Array.isArray(item.before)) {
    before = new Set(item.before);
  }
  let stage = 0;
  if (typeof item.stage === "number") stage = item.stage;
  let i = this.taps.length;
  while (i > 0) {
    i--;
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
```

`before` 可以是字符串（插件名）或者数组（由插件名组成），被初始化为 `Set` 结构的数据。在判断优先级之前，先对 `before` 进行处理：逐一判断当前项是否存在 `before` 中，如果存在就从 `before` 中删除并继续向前遍历；如果不存在但 `before` 不为空也继续向前遍历。

对于为什么要倒序遍历，因为大部分的插件并不会设置优先级，即默认优先级，所以应该从低优先级的右侧开始，这样就不需要遍历所有插件了。

顺便，再贴一下插件实现的接口：

```js
interface Tap {
  name: string,
  type: string
  fn: function,
  stage: number,
  before: string|array,
}
```

# 总结

钩子实例用于存放所有插件的 `tap` 数组，其实是一个优先级队列，优先级由 `state` 和 `before` 共同决定，优先级越高越靠前。

至此，钩子的基本内容就介绍完了，后续会逐个讲解不同类型的钩子，以及异步挂载方式。
