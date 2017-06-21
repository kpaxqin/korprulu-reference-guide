# Redux实践

## 初始化store

在入门示例中，我们使用了withRedux为页面初始化Redux。有Redux经验的开发者应该会感到奇怪，在社区常见的实践中，页面只需要connect，初始化Redux应该是在应用的最顶层。

这里是我们结合使用Redux/Reflux的经验和对Redux的理解，在实践上做的裁剪与变形，具体见：[Redux状态管理之痛点、分析与改良](https://zhuanlan.zhihu.com/p/27093191)

## 异步与请求

### 数据的权重

虽然异步数据都是从接口获取的，但它们的业务价值并不相同，在Web上我们可以将异步数据分为两类：

1. 主干数据：少了这些信息，这个页面的渲染就没有意义，通常页面的url也应该能够定位到主干信息
2. 次要数据：和主干信息无关的，只被某个组件需要的。比如一个异步的下拉框

下面是一个产品列表页的截图：

![product-list](http://oqt8yhdub.bkt.clouddn.com/product-list.png)

图中高亮的下拉框是异步的，但明显下方的product list列表才是页面的主干信息，列表的数据才是页面的主干数据。

> 如果是传统的面向客户(2C)的项目，主干数据通常意味着SEO的需求，因此必须放到服务端进行渲染。而次要的数据，完全可以放到客户端异步加载。这也是识别主干数据的另一种方式

相应的，处理这两类数据的技术方案是不一样的。

对于主干数据，建议在页面级去获取，再往下层组件传递。作为对比，理论上可以由使用数据的下层组件去获取，但我们不推荐这样做，因为本质上这些数据是页面的依赖而不是组件的依赖。

对于次要数据，需要考虑组件的复杂度、是否和业务相关，如果组件比较复杂、有可预见的复用需求，则考虑使用状态管理框架进行管理。

如果仅仅是异步下拉框，或是给已有组件添加异步数据依赖，一个[高阶组件](https://segmentfault.com/a/1190000004598113#articleHeader2)就足够解决问题。

```js
// 使用高阶组件给普通Select组件增加异步依赖
const CategorySelect = connectPromise({
  promiseLoader() {
    return categoryApi.getCatetories();
  },
  mapLoadingToProps(loading) {
    return {
      disabled: loading,
    };
  },
  mapDataToProps(data) {
    return {
      options: [
        { name: 'All', value: '' },
        ...data.items.map(({ name, id: value }) => ({ name, value })),
      ],
    };
  },
})(Select);
```

### ActionCreator方案

由于Redux官方没有直接提供异步支持，目前所有的异步处理都是基于中间件的第三方方案。

本项目使用[redux-action-tools](https://github.com/kpaxqin/redux-action-tools)作为Redux配套的异步库，详细的分析与思路见[Redux异步方案选型](https://zhuanlan.zhihu.com/p/24337401?group_id=850351553125711872)

总的来讲，redux-action-tools能够满足对异步方案的下列期望：

* 减少模板代码 
* 支持乐观更新 
* 低学习负担和依赖 
* 对异步的三个阶段(初始、成功、失败)提供聚类信息方便拦截

在接下来讲解loading和错误处理的章节中我们会更深入地使用和了解它。