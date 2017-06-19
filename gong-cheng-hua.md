# 工程化

本项目在工程化上主要使用如下技术：

* npm: 包管理
* Webpack: 打包
* Babel: 编译
* npm scripts: 脚本管理
* node-config: 环境变量管理

## 包管理
npm是目前社区最流行、资源最丰富的包管理工具，已经接近**事实标准**。

### Why not yarn?
[Yarn](https://yarnpkg.com)是facebook推出的基于npm资源的包管理工具，主要在如下两方面做了优化：

* 速度
* 默认版本锁

这两方面npm可以通过指定.npmrc和shrinkwrap做到。

而Yarn的缺点，在于它需要去跟随npm这一事实标准所带来的滞后性：

* 对npm参数支持有限
* 对npm script的支持不完全
* 对npm的变动有滞后性

这几方面一旦造成问题都是比较难绕过的。

长期来看，npm在版本发展中也必然会吸收Yarn的优点，最近的npm@5.0版本就是例子。因此，npm仍然是更优先的选择。

### Shrinkwrap

本项目使用了npm的[shrinkwrap](https://docs.npmjs.com/cli/shrinkwrap)锁定版本，并且强烈建议多数项目也使用它。

npm默认的install行为是[caret range](https://www.npmjs.com/package/semver#caret-ranges-123-025-004)，也就是说如果某个库的当前版本是`1.0.2`，install后在package.json中配置会是`^1.0.2`。如果你再次执行`npm install`的时候作者发布了`1.0.3`甚至`1.1.0`，就会下载该最新版本，而不是当初的`1.0.2`。

这种默认行为的背后是语义化版本[semantic-versioning](https://docs.npmjs.com/getting-started/semantic-versioning)，理论上只要不是版本号第一位(major version)的变更，就不应该有破坏性的行为。

然而这个假设非常理想，实际项目中很容易遇到

* 不严格遵守语义化版本的npm库
* 维护者也在patch/minor级版本上引入breaking bug

而这导致的问题，很可能是一个版本在开发或QA阶段正常，到生产环境却因为打包了另一个依赖版本出bug。

更麻烦的是，当项目大到一定程度后，依赖往往非常复杂。出问题的可能不是项目的直接依赖，而是依赖的依赖，非常难定位。

因此，版本锁对项目质量和稳定性的提升已经逐渐成为社区共识，被多数团队采用，最新的Yarn也将其作为默认行为。

## npm scripts

前端项目最常见的脚本需求有：

* 启动开发环境
* 打包
* 测试
* git钩子

我们使用 [npm scripts](https://docs.npmjs.com/misc/scripts) 完成这类需求：

```js
"scripts": {
  "precommit": "lint-staged",  // git commit 前置检查
  "prepush": "npm test", // git push 前置检查
  "start": "npm run start:dev & npm run start:mock", // 启动开发环境和mock数据
  "start:dev": "node server.js", // 启动开发环境
  "start:mock": "node ./mock/dyson.config.js", // 启动mock数据
  // 生产环境打包
  "build": "npm run clean && NODE_ENV=production webpack --config ./webpack/webpack.prod.config.js",
  // 清空打包输出目录
  "clean": "rimraf ./_dist",
  // 代码规范检查
  "lint": "eslint 'src/**/*.js?(x)'"
},
```

注：git钩子如`precommit`和`prepush`是由[husky](https://github.com/typicode/husky)提供的。

## 环境配置管理

在持续集成中，往往需要将项目部署到多个环境，不同的环境对应不同的配置(例如API地址)。典型的环境配置如下：

```js
// envs/default.json
{
  "basename": "/management-web", // 路由基础路径
  "server": { // API服务器配置
    "url": "http://localhost:3001/desktop/staff",
    "publicResource": "https://dev.korprulu.exmertec.com/rest"
  },
  "license": { // 第三方服务配置
    "gaodeMap": "f97efc35164149d0c0f299e7a8adb3d2",
    "gaodeService": "0552ca2fe891f7852ad1db80e9fde628"
  }
}

```

我们希望能够单独管理不同环境的配置文件，并且在运行时能正确地读取当前环境的对应配置。

本项目采用了[node-config](https://github.com/lorenwest/node-config)结合webpack的DefinePlugin实现配置注入。

示例代码如下(实际中本项目还定制了配置目录)：

```js
//webpack.config.js
const config = require('config')

//
const webpackConfig = {
  //...other configs
  plugins: [
    new webpack.DefinePlugin({
      'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV),
      'process.env.config': JSON.stringify(config)
    }),
  ]
}
```

通过上述代码，node-config会根据命令行的`NODE_ENV`、`HOST`等参数读取不同的配置文件，而webpack会把所有代码中的`process.env.config`替换成node-config读取到的配置。

> node-config读取规则见：https://github.com/lorenwest/node-config/wiki/Configuration-Files

这样只要在CI中配置不同的命令行参数，就可以实现指定配置。

通常envs文件夹下包含如下配置：

```
envs
├── default.json         // 默认配置，本地开发：npm run start
├── dev.json             // dev环境配置，本地开发：NODE_ENV=dev npm run start
├── dev-production.json  // dev环境配置，打包： HOST=dev npm run build
├── qa-production.json   // qa环境配置，打包： HOST=qa npm run build
└── production.json      // 生产环境配置，打包： npm run build
```

其中，所有打包用的配置文件名都包含production，这是因为`NODE_ENV=production`能确保各种webpack插件按照生产环境标准进行压缩等处理。不同的环境如qa/dev/production，则使用HOST环境变量进行区分。

基于这点，我们将`NODE_ENV=production`写进了npm scripts的build命令，打包时只需指定HOST即可。

## 代码版本追踪

由于打包后的代码被压缩和混淆，前端很难确认当前页面对应的代码版本。当功能与期望不符时，很难快速定位是实现问题还是部署问题。

通过多个项目的实践总结，在打包时读取当前git的commit hash，再将hash字符串注入页面的meta是非常好的解决方案。

```js
//webpack.config.js
const gitInfo = require('git-rev-sync');

const webpackConfig = {
  //...other configs
  plugins: [
    new HtmlWebpackPlugin({
      filename: 'index.html',
      template: './index.tpl.html',
      inject: false,
      COMMIT_HASH: gitInfo.long()
    }),
  ]
}
```

```html
<!-- index.tpl.html -->
<html>
  <head>
    <meta name="ci:build" content='build-hash:<%= htmlWebpackPlugin.options.COMMIT_HASH%>'>
  </head>
  <body><body/>
</html>
```

这样一来就可以非常方便地查看当前页面对应的代码版本：

![build-hash](http://oqt8yhdub.bkt.clouddn.com/build-hash.png)