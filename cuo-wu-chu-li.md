# 错误处理

错误处理，特别是针对API等异步操作的错误处理，是每个项目都会遇到的需求。

通常项目里都有一套比较通用的，适合多数场景的处理逻辑，比如弹窗提示错误信息。同时在一些特定场景下，又需要绕过通用逻辑进行单独处理，比如表单的异步校验。

![error-processing](http://oqt8yhdub.bkt.clouddn.com/error-processing.png)

总结起来有两个问题需要解决：

* 怎么**使用**通用处理逻辑
* 怎么**跳过**通用处理逻辑

## 反例集锦

经过多个项目的观察，常见的*不太理想*的处理手法

### 一、API层处理

	```js
	function fetchWrapper(args) {
		return fetch.apply(fetch, args)
			.catch(commonErrorHandler)
	}
	```
	
在较底层封装ajax库可以轻松实现通用的错误处理，但问题也非常明显：

1. **覆盖面有限**：错误处理**通常**针对API，但不意味着**只**针对API
2. **灵活性不足**：由于通用处理在底层，想跳过的话相关配置和逻辑需要穿透多层封装。
3. **不易组合**：比如有的场景一个用户操作需要多个异步请求，对用户而言操作只有一次，异常处理也只应该有一次，但在API层处理异常显然会重复触发通用处理函数

### 二、Action层调用错误处理

```js
	//action creator
	const getData = createAsyncAction(GET_DATA, function(id) {
		return api.getData(id)
			.catch(commonErrorHandler) //调用错误处理函数
	})
```
	
在有业务意义的action层调用通用处理逻辑，能按需调用，不妨碍异步请求的组合，处理的异步行为也不限于API请求。上一个方案的三大缺点都得到了很好的解决。

但是，通用处理往往适用于**多数**场景，这样写又引申出了两个新的问题：

1. 业务代码冗余：几乎每个action都得加上一段"默认"应该存在的代码
2. 风险增加：忘写这句代码就意味着没有任何错误处理的"裸奔"

### 三、监听错误Action

也有人把上面的方案做个依赖反转，改为在通用逻辑里监听业务action：
	
```js
	function commonErrorReducer(oldState, action) {
		switch(action.type) {
			case GET_USER_DATA_FAILED:
			case PUT_USER_DATA_FAILED:
			//... 茫茫多的其它action type
			return commonErrorHandler(action)
		}
	}
```
	
这样做的本质是把冗余从业务代码中拿出来集中管理。

问题在于**每添加一个请求，都需要修改公共代码**，把对应的action type加进来。且不说并行开发时merge冲突，如果加了一个异步action，但忘了往公共处理文件中添加——这是很可能会发生的——而异常是分支流程不容易被测试发现，等到发现，很可能就是事故而不是bug了。

它解决了方案二的第1点问题，却让第2点问题变得更加严重了。

## 基于元数据拦截

通过以上几种常见方案的分析，比较完善的错误处理需要具备如下特点：

1. 面向业务动作(action)，而非直接面向请求
2. 不侵入业务代码
3. 默认使用通用处理逻辑，无需额外代码
4. 可以绕过通用逻辑

异步行为天然有三种状态`开始`、`成功`或`失败`，分别对应到三个action，如果这些action能包含对应的状态信息，再在middleware层进行判断，过滤出失败状态的action进行处理是否可行？流程如下图：

![error-middleware](http://oqt8yhdub.bkt.clouddn.com/error-middleware1.png)

### Flux Standard Action

**怎么在action中包含状态信息呢？**

在社区，`redux`和`react`社区的活跃贡献者，现已加入react团队的[Andrew Clark](https://github.com/acdlite)就提出过[`flux standard action`](https://github.com/acdlite/flux-standard-action)(以下简称FSA)的概念试图为action定义一个更友好的结构规范：

> An action MUST
>
> * be a plain JavaScript object.
> * have a `type` property.
> 
> An action MAY
>
> * have an `error` property.
> * have a `payload` property.
> * have a `meta` property.
>
> An action MUST NOT include properties other than `type`, `payload`, `error`, and `meta`.

这个规范相当简单，仅仅是对action添加了结构约束。

借助其中的meta属性，我们可以存取action的元数据——这完美契合了我们的需求，异步行为的状态是对相关action本身的描述，毫无疑问属于`meta data(元数据)`。

### 定制Meta

有了统一的、规范的meta，我们可以在异步每个状态的action上添加相应的meta，由middleware根据meta进行判断，再派生出新的action去触发通用的错误处理逻辑。

如果我们把"添加相应的meta"作为默认行为，就完美做到了`不侵入业务代码`、`基于action`和`默认使用通过处理`三点。可是特殊场景怎么办？如何绕过默认的逻辑呢？

既然middleware层是根据meta拦截，只要再往meta中添加信息，告诉middleware"*这是特例，请跳过*"就好了，代码如下：

```js
function errorMiddleWare({dispatch}) {
  return next => action => {
    const asyncStep = _.get(action, 'meta.asyncStep');
    const omitError = _.get(action, 'meta.omitError'); //是否跳过

    if (!omitError && asyncStep === ASYNC_PHASES.FAILED) {
      dispatch({ // 触发通用错误处理
        type: 'COMMON_ERROR',
        payload: {
          action
        }
      })
    }
    
    return next(action);
  }
}
```

### Action creator

以上基于元数据拦截的思路理论上虽然可行，却对创建异步相关action提出了两个要求：

* 默认添加异步行为的相关meta
* 支持对meta定制化

这两点我们使用的redux-action-tools都提供了支持，它按下表提供默认的meta:

|     type           | When         |  payload  | meta.asyncPhase    |
| --------           |  -----      | :----:    | :----:  |
| `${actionName}` | before promiseCreator been called | sync payload | 'START' |
| `${actionName}_COMPLETED` | promise resolved | value of promise | 'COMPLETED' |
| `${actionName}_FAILED` | promise rejected | reason of promise | 'FAILED' |

同时支持meta的定制，以下两个请求分别对应使用默认错误处理和自定义错误处理：

```js
import { ASYNC_PHASES } from 'redux-action-tools'

const defaultAction = createAsyncAction(
  type,
  promiseCreator,  
  // 不提供第三个参数metaCreator时，使用默认的meta信息
) 

const customizedAction = createAsyncAction(
  type, 
  promiseCreator, 
  (payload, defaultMeta) => {
    return { ...defaultMeta, omitError: true }; //向meta中添加配置参数
  }
)
```

### 通用错误处理

在上面errorMiddleWare的代码中，我们使用了`COMMON_ERROR`来触发通用错误处理，接下来的流程就是标准的用reducer响应action并修改状态，这是标准的redux流程，用它作演示是因此它最清晰、耦合度最低。

实际项目中，也可以视项目情况，直接将部分逻辑放在errorMiddleware中：

```js
function errorMiddleWare({dispatch}) {
  return next => action => {
    const asyncStep = _.get(action, 'meta.asyncStep');
    const omitError = _.get(action, 'meta.omitError'); //是否跳过

    if (!omitError && asyncStep === ASYNC_PHASES.FAILED) {
      const { payload: error } = action;

      if (error.response && error.response.status === 401) {
        userStorage.removeUser()
          .then(() => dispatch(routerActions.replace({
            path: '/sign-in',
            query: { reason: '401' },
          })));
      } else {
        dispatch(modalActions.showToastAlert({
          text: getErrorMessage(error), //提取错误信息
          isError: true,
          timeout: 3000,
        }));
      }
    }
    
    return next(action);
  }
}
```

需要注意getErrorMessage函数，在通用错误处理逻辑中，不可避免的会有类似的函数，从经验上这个函数最好既能接受api请求错误，也能处理普通的Error对象：

```js
export const getErrorMessage = error => (
  error.responseBody
    ? getMsgFromApiCode(error.responseBody.errorCode) // from API
    : (error.message || 'Unkown error') // from Error object
);

```

这样的设计可以支撑**轻量**的定制——使用通用错误处理的流程，只定制错误信息：

```js
const action = createAsyncAction(
  type,
  ()=> getData().then(
    data=> data,
    error=> Promise.reject(new Error('Custom message')),
  ), 
) 
```

### 自定义错误处理

