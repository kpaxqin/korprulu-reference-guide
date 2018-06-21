# Korprulu前端参考手册

这里是Korprulu后台管理系统前端的参考手册，主要解释了本项目在代码风格、工程化、常见场景采用的技术方案和背后的原因。

## 目录结构

* `src`: 源代码
  * `.jsx`: 包含jsx语法的文件,  
  * `.less`: 样式文件
  * `__tests__`: 单元测试文件夹
    * `.specs.js(x)`: 单元测试
* `test`: 运行测试的基础设施
* `mock`: 前端Mock数据脚本
* `webpack`: webpack 配置 
* `_dist`: 打包输出路径
* `coverage`: 测试覆盖率路径

## 技术方案

* [es2015](https://babeljs.io/learn-es2015/) + [stage-1](https://babeljs.io/docs/plugins/preset-stage-1/) + JSX : 语言规范
* [React](https://facebook.github.io/react/) : 视图层
* [Redux](reduxjs.org) : 状态管理
* [Lodash](lodash.com/docs) : 工具函数库
* [Less](http://lesscss.org/) : CSS预处理
* [bootstrap](getbootstrap.com) : 基础样式
* [React-Router](https://reacttraining.com/react-router/) : 路由 **可能替换**
* [Dyson](https://github.com/webpro/dyson) : API数据Mock
* [Webpack](webpack.github.io) : 打包工具
* [React-bootstrap](react-bootstrap.github.io) : 基础组件库

## 概述

### 基础架构

![architechture](http://oqt8yhdub.bkt.clouddn.com/arch.png)

_架构图_

本项目使用React作为视图层库，React在社区和公司内都有较广泛的群众基础，学习和推广成本都较低。

使用React-bootstrap作为基础组件库，样式上也用了bootstrap的第三方主题。这是考虑到Bootstrap能够在保证较高UI风格一致性的前提下加快开发效率，且有大量第三方主题方便各项目组定制。

同时使用Redux作为状态管理框架，它在提倡纯函数、强可预测性方面有显著的竞争力，上图`Page`-&gt; `Action creator`-&gt; `Action`-&gt; `store`即是其单向数据流思想的体现\(虚线圆环\)。

使用React-Router作为路由库，但抛弃了官方推荐的`路由即组件`，转而采用了扁平、集中的路由，如上图所示，在架构中路由是完全独立的一部分，不侵入组件代码，仅仅承担url到Page的转发作用。

### 代码结构

```
src
├── store : 店铺业务模块
│   ├── actions : *action目录，可选，可为单文件
│   ├── components : 组件目录
│   │   ├── StoreList.jsx : 业务组件
│   │   └── StoreList.less : 业务组件对应样式，同文件名不同后缀
│   ├── constants : *常量目录，可选，可为单文件
│   ├── pages : 页面
│   │   ├── List.jsx : 列表页
│   │   └── Detail.jsx : 详情页
│   ├── reducers : *reducer目录，可选，可为单文件
│   └── index.js  业务模块的入口文件，默认导出所有pages
├── shared : 通用组件、工具文件夹
├── routes.jsx : 路由配置
├── index.tpl.html : 项目html模板
└── index.jsx : 项目入口文件
```



