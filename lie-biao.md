# 列表页

列表页通常包含了`搜索`、`翻页`等要素，这些要素组合在一起，既是一套可复用的行为模式，也是**url可用性**的重灾区。

有很多开发者在使用SPA开发列表视图时，有意或无意地忽略**url**的重要性，典型的例子如[react+redux的realworld example](https://react-redux.realworld.io/#/?_k=v9l3hb)，当翻页后再刷新\(或前进、后退\)，页码信息就会丢失，这会带给用户非常差的体验。

> 相关阅读：[WebApp防坑手册](https://segmentfault.com/a/1190000005864691?_ea=2057859)

## 设计与原则

要让页面刷新、分享后可用，就要保证url中有足够完备的信息从api获取数据，而列表中常见的`搜索`、`翻页`、`排序`，都属于url中query部分的职责，因此，第一步就是把这些信息放进query中，即`/list?pageIndex=1&name=foo`。当在浏览器中访问这个链接时，页面会根据query信息渲染搜索栏，同时请求api，最后渲染api返回的列表数据。

而当用户使用搜索或翻页时，应该首先修改url，然后重复上述流程，这样使得整个过程能够**单向**化——由url驱动页面的渲染。

总结成流程图如下所示：

![list-arch](http://oqt8yhdub.bkt.clouddn.com/list-arch.png)

## reduxList

遵循上述结构，结合类似`redux-form`的封装思路，列表视图的行为也是可以被封装的。

```js
const ListPage = (props) => {
  const {
    listData, // 列表数据
    searchParams,  // 查询参数
    sortParams, // 排序参数
    searchList, // 查询列表
    sortList, // 排序列表
    goToPage, // 翻页
  } = props;
  return (
    <div>
      <SearchBar searchParams={searchParams} onSearch={searchList} />
      <List data={listData} onSelectPage={goToPage} /> 
    </div>
  );
} 

export default connectListView({
  name: 'example-list', // 列表的唯一标识
  dataLoader: api.getListData, // 返回Promise<ListData>的函数，其第一个参数为mapLocationToRequest函数转换出的request对象
})(ListPageComponent);
```

流程图上标注出了三个关键转换节点：

1. mapLocationToSearch
2. mapLocationToRequest
3. mapSearchToQuery

借助它们我们可以实现各种定制化的需求：

## 默认查询参数

如果按上面的流程图，整个界面由url驱动，那么url中的query与搜索栏应该呈一一对应关系，然而实际中默认参数的需求很常见，如下图：

![pre-selected](http://oqt8yhdub.bkt.clouddn.com/store-product-pre-select.jpeg)

图中页面要求红色箭头所指的`status`选项卡默认选中`Active`，而此时红框所示的url没有任何query。当然用户也可以通过点击选项卡在`Active`和`Inactive`间切换。

分解这个问题，我们可以得到几个要素：

* 当url中没有status值时，按选中`Active`渲染搜索拦
* 当url中没有status值时，按选中`Active`发起列表数据请求
* 当url中有status值时，按该值渲染搜索栏并发起列表数据请求

对应到上面列出转换节点，我们需要定制其中的：

* mapLocationToSearch: 当url中没有status值时，转换出的search参数应带有`status: Active`，否则按默认行为处理
* mapLocationToRequest: 当url中没有status值时，转换出的请求参数应带有`status: Active`，否则按默认行为处理

对应到代码：

```js
import {connectListView, DEFAULT_LIST_CONFIG} from 'redux-listview'

export default connectListView({
  name: 'example-list', 
  dataLoader: api.getListData, 
  mapLocationToRequest(location) {
    const defaultRequest = DEFAULT_LIST_CONFIG.mapLocationToRequest(location);

    return { status: 'ACTIVE', ...defaultRequest };
  },
  mapLocationToSearch(location) {
    const defaultSearch = DEFAULT_LIST_CONFIG.mapLocationToSearch(location);

    return { status: 'ACTIVE', ...defaultSearch };
  }
})(ListPageComponent);
```

上述对`mapLocationToSearch`和`mapLocationToRequest`的定制都是通过简单的函数组合实现的，非常简单直接。

可能有人要问这两个函数高度相似，为什么不想办法合并？事实上这种相似只是上述场景下的特例，还有其它的场景，比如对请求数据做格式转换，就只会用到`mapLocationToRequest`：

```js
mapLocationToRequest(location) {
  const defaultRequest = DEFAULT_LIST_CONFIG.mapLocationToRequest(location);

  // createTime = '2015-01-01,2017-12-12'
  const { createTime, ...others } = defaultRequest; 

  // createTimeFrom = '2015-01-01'; createTimeTo = '2017-12-12'
  const { from: createTimeFrom, to: createTimeTo } = transformTimePair(createTime); 

  return {
    ...others,
    createTimeFrom,
    createTimeTo,
  }
}
```

这些场景有可能相互交错、同时发生，对这种难以预见的、多变的需求，设计基础接口时最好偏向其普适性，而不是追求固定场景下的便利。

而这些不太便利的场景，比如上面遇到的代码重复，可以通过提取工具函数解决。





更新：

对`mapLocationToSearch`和`mapLocationToRequest`的代码重复问题，最近也在思考通过抽象成 

`mapLocationToSearch`和`mapSearchToRequest`的方式解决，这样在默认值场景下代码重复度更低，且抽象意义也可以接受。



另目前看来，search参数不应该以 `/list?pageIndex=1&foo=b` 这样简单的方式放到url上，因为页面一旦有其它url params，我们就无法识别，而search行为是需要完整重围url上的列表查询参数的。建议的方向是将search对象序列化后放到query上：

 `` `/list?listQueries=${encodeURIComponent(JSON.stringify({pageIndex:1, foo:'b'}))}```

