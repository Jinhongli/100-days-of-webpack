# 运行 `webpack` 01

下载 webpack 包之后，我们的 `node_modules` 文件夹的 `.bin` 文件夹下就会出现 webpack 的执行文件：`webpack`。对于Unix系统来说，我们可以直接指定文件的执行环境是 `node`，但是对于 Windows 来说，需要使用脚本运行真正的执行文件`./node_modules/webpack/bin/webpack.js`。

先来大致看一下入口文件的结构：

```javascript
#!/usr/bin/env node

process.exitCode = 0;

// 执行命令
const runCommand = (command, args) => {
  // 开启子线程执行命令
};

// 验证是否安装
const isInstalled = packageName => {
  // 根据 require.resolve 验证包是否安装
};

// 命令行工具
const CLIs = [
  // 0 webpack-cli
  // 1 webpack-command
];

// 已经安装的命令行工具
const installedClis = CLIs.filter(cli => cli.installed);

if (installedClis.length === 0) {
  // 如果一个也没有安装，打印提示，并让用户选择其中一个下载，下载完成之后再加载
} else if (installedClis.length === 1) {
  // 如果已经存在一个命令行工具，加载对应工具
} else {
  // 如果两个都有，打印警告
}

```
大致流程就是：

1. 执行该文件的执行环境，已经默认的执行结果；
2. 声明两个辅助函数；
3. 声明 `webpack` 支持的命令行工具，并查找已经安装的命令行工具；
4. 根据已安装的命令行工具个数分别：提示下载、加载 cli、打印警告；

# 总结

当我们运行 `webpack` 这个命令（运行 webpack 执行文件），其作用是加载命令行工具。