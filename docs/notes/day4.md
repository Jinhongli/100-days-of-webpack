# Tapable 架构 01

上一节已经讲到了使用 `Compiler` 构造器来生成编译器实例，这就引出了 webpack 核心架构，Tapable。因此我们得先了解所谓的 Tapable 到底是什么。

一个 Tapable 的实例其实就是实现了一些钩子的对象，其目的是为了把实例的功能进行解藕：以插件的方式将实例的各个功能分离，然后再把这些功能插件挂载特定钩子上，然后统一调用（这看起来非常像我们的订阅 - 发布模式：只要插件订阅事件后，实例只需发布事件即可）。 Tapable 内部为我们封装了各种钩子，包括：

- SyncHook
- SyncBailHook
- SyncWaterfallHook
- SyncLoopHook
- AsyncParallelHook
- AsyncParallelBailHook
- AsyncSeriesHook
- AsyncSeriesBailHook
- AsyncSeriesWaterfallHook

这些所有的钩子类都继承自同一个基类： `Hook`。我们从最简单的 `SyncHook` 钩子类开始，首先演示如何使用同步钩子。

首先是创建钩子，以一个汽车为例：

```javascript
import { SyncHook } from 'tapable';

class Car {
  constructor() {
    // 声明汽车实例上的钩子
    this.hooks = {
      break: new SyncHook(),
    }
  }
  break() {
    // 调用挂载在 `break` 钩子上的所有函数
    this.hooks.break.call();
  }
}

const myCar = new Car();
```

现在我们有一辆 Tapable 的车子了，它目前只有一个钩子：`break`。这是一个同步钩子，但现在这个钩子是空的，上面什么都没有。

想要在钩子上挂载函数的话，使用 `Hook.prototype.tap(name, fn)` 方法。第一个参数 `name` 表示钩子函数的名称，第二个参数 `fn` 才是要挂载在钩子上的回调函数。

```javascript
myCar.hooks.break.tap("WarningLampPlugin", () => {
  console.warn('Taillight is on.');
});

myCar.break(); // "Taillight is on."
```

但这个钩子有点无聊，有时候我们想要获取触发钩子时携带的参数，比如说设置汽车的速度时，插件应该会需要知道速度是多少：

```javascript
class Car {
  constructor() {
    // 声明汽车实例上的钩子
    this.hooks = {
      break: new SyncHook(),
      accelerate: new SyncHook(['newSpeed']), // 具有一个形参
    }
    this.speed = 0;
  }
  break() {
    this.hooks.break.call();
  }
  setSpeed(newSpeed) {
    this.speed = newSpeed;
    // 调用挂载在 `accelerate` 钩子上的函数，并传递实参
    this.hooks.accelerate.call(newSpeed);
  }
}

const myCar = new Car();
myCar.hooks.accelerate.tap("LoggerPlugin", newSpeed => {
  console.log(`Accelerating to ${newSpeed}`);
});

console.log(`Car's current speed is ${myCar.speed}`); // "Car's current speed is 0"
myCar.setSpeed(10);                                   // "Accelerating to 10"
console.log(`Car's current speed is ${myCar.speed}`); // "Car's current speed is 10"
```

在创建钩子实例的时候，可以以数组的形式设置钩子传递的形参（上面的代码中对 `accelerate` 钩子挂载 `LoggerPlugin` 时，设置的形参 `newSpeed` 与创建时传入的 `['newSpeed']` 的第一项一样，但这并不是必须的。事实上钩子函数外层会使用 `new Function()` 包装一层函数，`['newSpeed']`中的各项会作为这个函数的形参）。

# 总结

本节简单介绍了如何使用一个同步的 `SyncHook` 来实现一个 Tapable 的汽车类。通过这个例子我们知道了：

- 直接在实例上声明 `hooks` 属性来存放所有的钩子；
- 所有的钩子实例都继承自 `Hook` 实例；
- 通过继承自钩子基类的 `Hook.prototype.tap()` 方法把回调函数挂载在钩子上；
- 通过钩子实例上的 `call()` 方法，执行钩子上的所有回调函数，并且可以传递参数，需要在生成钩子实例时设置形参；