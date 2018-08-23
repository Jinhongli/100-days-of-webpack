# 运行 `webpack` 02

我们选择安装的 CLI 工具是 `webpack-cli`。在上一节的 `webpack.js` 文件中，检测通过 `require()` 直接加载工具会根据  `package.json` 中的的 `bin` 字段，加载 `node_modules/webpack-cli/.bin/cli.js` 文件，这个文件是用来定义 CLI 功能、解析参数，最后生成配置项，执行打包程序。其大致结构是：

```javascript
#!/usr/bin/env node

(function() {
  // 熟悉的 IIFE，为了能够使用 return

  // 使用本地版本的 webpack-cli 工具，本地没有安装的话，直接退出
  const importLocal = require('import-local');
  if (importLocal(__filename)) {return;}

  // 加载模块时，使用 V8 代码缓存
  require("v8-compile-cache");

  // 用于格式化错误信息
  const ErrorHelpers = require("./errorHelpers");

  // 非 webpack 打包的命令，比如 init 初始化一个 webpack 工程等
  // 如果有这种命令，就加载 `./prompt-command` 模块来执行，并直接退出，以后再详细说明
  const NON_COMPILATION_ARGS = [];
  const NON_COMPILATION_CMD = process.argv.find();
  if (NON_COMPILATION_CMD) {}

  // 使用 yargs 定义命令行
  const yargs = require("yargs").usage(
    //...
  );

  // 配置命令行
  // webpack 打包配置项，就是 webpack.config.js 中会出现的字段
  require("./config-yargs")(yargs);
  yargs.options({
    // 命令行配置项
  });

  // 手动执行 yarg 解析命令行参数
  // yarg 会在查看帮助或版本的时候提前终止进程，导致较长的帮助文本会被截断
  yargs.parse(process.argv.slice(2), (err, argv, output) => {

    // 无关紧要的配置

    // 解析 CLI 传入的参数得到 webpack 配置项
    try {
      options = require("./convert-argv")(argv);
    } catch (err) {
      // 如果出错，使用 ErrorHelpers 格式化错误信息
    }

    function ifArg(name, fn, init) {
      // 辅助函数，策略执行
    }

    function processOptions(options) {
      // 先对配置进行预处理（添加默认配置），以后再具体说明
      // ...

      // 然后加载 webpack 模块，执行打包
      const webpack = require("webpack");
      let lastHash = null;
      let compiler;
      try {
        // 打包成功，得到编译器对象
        compiler = webpack(options);
      } catch (err) {
        // 打包失败，打印错误信息
      }
    }

    // 使用配置进行打包
    processOptions(options);
  })
})();
```

大致流程是：
1. 加载辅助工具（本地版本、模块缓存、错误格式化）；
2. 对非打包的命令执行其他脚本（`webpack init`: 初始化 webpack 工程等）；
3. 配置 `yargs`，包括打包默认配置项和命令行配置项；
4. 执行 `yargs`，将命令行参数转化，生成最终的打包配置项（`options` 对象）；
5. 通过 `processOptions` 加载 `webpack` 得到编译器
6. 使用 `compiler.run` 执行打包程序；

# 总结

当命令行工具加载完毕之后，通过执行 `yargs` 解析命令行，生成最终的 webpack 打包配置项，然后加载 `webpack` 包，执行打包程序，得到编译器对象以及编译结果。
