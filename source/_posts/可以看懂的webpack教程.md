---
title: 可以看懂的webpack教程
date: 2020-04-26 17:18:01
tags: 技能篇
---

关于webpack，真是让人又爱又恨。作为前台端水的一员，搞懂webpack是必不可少的！
Webpack可以看做是模块打包机：它做的事情是，分析你的项目结构，找到JavaScript模块以及其它的一些浏览器不能直接运行的预编译语言（scss，TypeScript等），并将其打包为合适的格式以供浏览器使用。

一份webpack配置（`webpack.config.js`）包含以下内容:
+ Entry：**入口**，Webpack 执行构建的第一步将从 Entry 开始，可抽象成输入。
+ Module：**模块**，在 Webpack 里一切皆模块，一个模块对应着一个文件。Webpack 会从配置的 Entry 开始递归找出所有依赖的模块。
+ Chunk：**代码块**，一个 Chunk 由多个模块组合而成，用于代码合并与分割。
+ Loader：**模块转换器**，用于把模块原内容按照需求转换成新内容。
+ Plugin：**扩展插件**，在 Webpack 构建流程中的特定时机注入扩展逻辑来改变构建结果或做你想要的事情。
+ Output：**输出结果**，在 Webpack 经过一系列处理并得出最终想要的代码后输出结果。

## 从零开始配置
### 首先，当然是需要安装webpack工具
```
 npm init --yes // 生成一份package.json文件，方便管理版本
 npm i webpack webpack-cli --save-dev // 安装webpack
```

有了 Webpack 后，就可以直接运行 webpack 命令来打包 JS 模块代码
```
 npx webpack
```
这个命令在执行的过程中，Webpack 会自动从 `src/index.js` 文件开始打包，然后根据代码中的模块导入操作，自动将所有用到的模块代码打包到一起。完成之后，打包后的文件会出现在`dist`文件夹里的`main.js`文件

+ **一个小tips：让vs code支持webpack智能提示**
我们通过 import 的方式导入 Webpack 模块中的 Configuration 类型，然后根据类型注释的方式将变量标注为这个类型，这样我们在编写这个对象的内部结构时就可以有正确的智能提示了
```javascript
import { Configuration } from 'webpack'
/**
 * @type {Configuration}
 */

// 或者是这种写法
/** @type {import('webpack').Configuration} */

module.exports = {
    // some config
}
```
**注意：在配置完成后记得注释掉这段import代码，否则编译会出错。因为node环境还不支持import语句**

### 可以开始自定义配置webpack了

大多数情况下，我们都会有一些自定义的需求，因此，可以新建一个`webpack.config.js`文件来自定义配置webpack。
webpack.config.js 是一个运行在 Node.js 环境中的 JS 文件，也就是说我们需要按照 CommonJS 的方式编写代码，这个文件可以导出一个对象，我们可以通过所导出对象的属性完成相应的配置选项。
```javascript
module.exports = {
    // some config
}
```
####  自定义入口文件(`entry`)和输出结果(`output`)

```javascript
const path = require('path')

module.exports = {
   entry: './src/main.js', // @入口文件 string | object | array
   output:{                // @输出选项：
        filename: '[name].[hash].js', // 文件名： string
        path: path.join(__dirname, 'output'), // 所有输出文件的出口目录
        publicPath: "/assets/", // 构建文件的输出目录
   },
   module:{},
   plugins:[],
   devServer:{}
}
```

