# 运行 `webpack` 03

到了现在，我看终于看到了 `compiler = webpack(options);` 这个语句，终于到了主角登场的时间。通过 webpack 包的 `package.json` 的 `main` 字段可以找到入口文件 `node_modules/webpack/lib/webpack.js`，然后来看一下 `cli.js` 中的 `webpack(options)` 内部都干了什么吧。

```javascript
'use strict';

// 加载依赖
const Compiler = require('./Compiler');
const MultiCompiler = require('./MultiCompiler');
const NodeEnvironmentPlugin = require('./node/NodeEnvironmentPlugin');
const WebpackOptionsApply = require('./WebpackOptionsApply');
const WebpackOptionsDefaulter = require('./WebpackOptionsDefaulter');
const validateSchema = require('./validateSchema');
const WebpackOptionsValidationError = require('./WebpackOptionsValidationError');
const webpackOptionsSchema = require('../schemas/WebpackOptions.json');
const RemovedPluginError = require('./RemovedPluginError');
const version = require('../package.json').version;

const webpack = (options, callback) => {
  // 一些验证配置项的功能，先忽略掉...

  // 声明最终的编译器对象
  let compiler;
  if (Array.isArray(options)) {
    // 多个配置项，递归得到多个编译器对象
    compiler = new MultiCompiler(
      options.map(options => webpack(options))
    );
  } else if (typeof options === 'object') {
    // 单个配置项
    // 添加默认配置
    options = new WebpackOptionsDefaulter().process(options);
    compiler = new Compiler(options.context);
    compiler.options = options;
    // 执行 plugin 的 apply 方法（讲对应功能挂载再编译器或者编译结果的钩子上）
    new NodeEnvironmentPlugin().apply(compiler);
    if (options.plugins && Array.isArray(options.plugins)) {
      for (const plugin of options.plugins) {
        plugin.apply(compiler);
      }
    }
    // 执行编译器环境钩子
    compiler.hooks.environment.call();
    compiler.hooks.afterEnvironment.call();
    // 根据配置项预处理编译器，并得到最终的配置项
    compiler.options = new WebpackOptionsApply().process(
      options,
      compiler
    );
  } else {
    throw new Error('Invalid argument: options');
  }
  if (callback) {
    // 如果没有使用配置文件的方式，而是直接引入 webpack 手动执行打包，可以使用回调的方式
  }
  // 返回编译器对象
  return compiler;
};

// 定义模块导出
exports = module.exports = webpack;
exports.version = version;

// 之后就是各种 webpack 内置插件的导出
```

大致流程：

1. 添加默认配置项
2. 生成编译器实例
3. 执行所有插件，全部挂载到对应的钩子上。
4. 执行编译器环境钩子
5. 根据配置项预处理编译器，并得到最终的配置项
6. 返回编译器实例

# 总结

其实这个 `webpack.js` 文件的作用就是验证、处理传入的配置项、挂载插件、预处理编译器之后，得到最终的编译器和配置项。