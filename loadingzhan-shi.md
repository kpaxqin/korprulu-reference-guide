# Loading展示

对于异步行为，Loading是另一种常见的需求，它承担两个作用：

* 提示用户异步行为正在进行
* 阻止用户重复操作

通常，为了避免每个页面、场景单独设计，很多项目会使用较大范围(覆盖整个页面或功能区)的遮罩作为通用Loading，供多数异步行为默认使用。

而对交互和体验要求较高场景，则不使用大范围的Loading，按设计要求细化处理，比如disable某个按钮。

分析到这里，其实情况和错误处理已经非常类似了：需要一套默认启用的通用方案，同时要提供跳过通用处理的机制，能方便地进行自定义处理。

## 通用Loading展示

对于通用Loading来说，处理的大体思路和错误处理是一样的，即用中间件检查元数据：

![loading-middleware](http://oqt8yhdub.bkt.clouddn.com/loading-middleware.png)

对应代码如下：

```js
export function loadingMiddleWare({ dispatch }) {
  return next => (action) => {
    const asyncPhase = _.get(action, 'meta.asyncPhase');
    const omitLoading = _.get(action, 'meta.omitLoading');

    if (asyncPhase && !omitLoading) {
      dispatch({
        type: asyncPhase === ASYNC_PHASES.START
          ? actionTypes.ASYNC_STARTED
          : actionTypes.ASYNC_ENDED,
        payload: {
          source: 'ACTION',
          action,
        },
      });
    }

    return next(action);
  };
}
```

由于loading的展示通常不涉及副作用，这部分代码更加简洁，只需要派发相关的action去触发通用loading的展示逻辑。

类似错误处理中的omitError，loadingMiddleware也提供了omitLoading来跳过通过的loading展示。

特别需要注意的是，在定制meta时，通过删除或修改`meta.asyncPhase`同样能达到跳过通过处理的效果，但这会导致一损俱损——所有依赖`meta.asyncPhase`的中间件都会受影响。因此，无论是loadingMiddleware还是errorMiddleware，"跳过"配置都被设计为增量属性。

## 自定义Loading展示

和错误处理一样，自定义的部分只需要标准的redux工作流即可。