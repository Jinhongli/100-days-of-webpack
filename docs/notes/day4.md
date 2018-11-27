# Tapable 架构 01

上一节已经讲到了使用 `Compiler` 构造器来生成编译器实例，在深入讲解 Webpack 之前，我们需要先了解下 webpack 核心架构，Tapable，到底是什么玩意儿 🤔。

如果将 tap 翻译为窃听、偷听的话，那么 tapable 的大致意思就是将目标对象变为可窃听的。比如在你的电话中放入窃听器，这样我就能在**你说话的时候**做一些你不知道的事情。开个玩笑，应该是你手动在电话中放一个窃听器，然后让你的秘书在你与他人讲话的时候，帮你记录下你的日程。为了更加方便在计算机领域解释，后面会使用插件，钩子等术语代替窃听器。

所以 Tapable 就是将目标对象变为可“窃听”的，它包含了各种各样的钩子，通过实例化（通常是核心功能的生命周期）钩子并挂载对应的功能单元，实现模块化/插件化开发。

包含的钩子有：

- SyncHook
- SyncBailHook
- SyncWaterfallHook
- SyncLoopHook
- AsyncParallelHook
- AsyncParallelBailHook
- AsyncSeriesHook
- AsyncSeriesBailHook
- AsyncSeriesWaterfallHook

这些钩子都是同一个基类的派生类，`Hook`。下面是 `Hook` 实例的轮廓：

```js
class Hook {
  /******* 这些是“共有”的属性 & 方法 *******/ 
  /***************************************/ 
  // {[]} 实例属性；已挂载的所有插件列表
  taps;
  // {fn} 实例方法；执行所有插件
  call();
  // {fn} 实例方法；异步执行所有插件
  callAsync();
  // {fn} 实例方法；异步执行所有插件，返回 <Promise>
  promise();
  // {fn} 原型方法；挂载插件
  tap();
  // {fn} 原型方法；挂载异步插件
  tapAsync();
  // {fn} 原型方法；挂载 <Promise> 异步插件
  tapPromise();
  // {fn} 原型方法：编译执行函数
  compile();
  // {fn} 原型方法；查询是否存在插件
  isUsed();

  /******* 这些是“私有”的属性 & 方法 *******/ 
  // {[]} 钩子触发时传递给插件的变量列表
  _args;
  // {fn} 实例方法；编译所有插件后得到的执行函数
  _x;
  // {fn} 原型方法；将插件放置在 taps 内
  _insert();
  // {fn} 原型方法；执行编译函数
  _createCall();
  // {fn} 原型方法；重置编译函数
  _resetCompilation();
  // {fn} 原型方法；对特定插件注册拦截器
  _runRegisterInterceptors();

  /******* 高级功能，可以先略过不看 *******/ 
  // {[]} 实例属性；窃听器拦截器列表；这算是高级功能，可以先不用关
  interceptors；
  // {fn} 原型方法；对所有插件注册拦截器
  intercept();
  // {fn} 原型方法；添加执行函数的配置
  withOptions();
}
```

这样看起来不太容易理解，我们来举个汽车的例子：

```javascript
import { SyncHook } from 'tapable';

class Car {
  constructor() {
    this.hooks = {
      break: new SyncHook(),
    }
  }
  break() {
    this.hooks.break.call();
  }
}

const myCar = new Car();
```

首先 `Car` 类在实例化时会对实例设置一个 `break` 的同步钩子，用于监听刹车事件。当汽车调用 `Car.ptototype.break()` 方法时，使用 `newCar.call()` 来同步触发这个钩子上的所有插件。当然，现在这个钩子是空的，上面什么也没有。

假设现在我们要对 `Car` 类增加刹车灯的功能，并且想要将这个功能独立于 `Car` 类，我们可以在 `break` 钩子上挂载一个插件来实现：

```javascript
myCar.hooks.break.tap("WarningLampPlugin", () => {
  console.warn('Taillight is on.');
});

myCar.break(); // "Taillight is on."
```

`SyncHook.ptototype.tap(name<string>|options<object>, fn<function>)` 这个方法就是用来挂载同步钩子的函数，这个函数接受两个参数：第一个用于设置插件属性；第二个就是钩子触发时插件的回调函数。

现在一个 Tapable 的汽车就实现了，虽然比较简陋，但是包含了基本的：钩子、插件、触发操作。

接下来我们可以对这个简陋的汽车继续扩展功能，比如说加速与速度控制台功能。假设速度以及加速函数由实例来保存，而速度控制台需要插件来单独维护，那么显然需要实例将当前速度值发送给插件，这就涉及到插件参数：

```javascript
class Car {
  constructor() {
    this.hooks = {
      break: new SyncHook(),
      accelerate: new SyncHook(['newSpeed']), // 带有一个形参
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

注意在实例化 `accelerate` 这个钩子时，传入了 `['newSpeed']` 这个参数，用于显示的指明触发插件时，执行函数的形参。之所以这样是因为执行函数是由 `new Function()` 创建的，需要指明函数体内的形参。

接下来，在使用 `newCar.call()` 时，将当前值作为实参传入就可以了。

# 总结

本节简单介绍了 `Hook` 钩子实例的轮廓，并用 `SyncHook` 实现 Tapable 汽车类的例子演示了使用的工作流。通过这个例子我们知道了：

- 在目标对象上声明 `hooks` 属性来存放/管理所有的钩子；
- 通过钩子原型上的 `tap()` 方法把插件挂载在钩子上；
- 通过钩子实例上的 `call()` 方法，调用钩子执行函数。并且可以传递参数（需要在实例化钩子时设置形参）；

涉及名词解释：

- 目标对象：需要实现 tapable 功能的对象。
- 钩子：由目标对象自定义，并在特殊时机触发的额外属性/对象，用于管理插件挂载、执行。
- 插件：挂载在钩子上的功能单元。
- 插件回调函数：目标对象触发插件所处的钩子时，该插件的回调函数。
- 钩子执行函数：目标对象触发钩子时的回调函数，这个函数由所有插件的回调函数编译而成。