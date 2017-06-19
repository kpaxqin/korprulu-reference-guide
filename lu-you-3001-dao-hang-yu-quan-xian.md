# 路由与导航

## 路由配置
本项目目前采用了react-router作为路由库，目前是v4，使用它的初衷是跟随社区的主流实践。然而在使用过程中，结合使用react-router以往版本的经验，我们逐渐认为把组件库与路由库深度耦合并不是一个好的方向，这点社区也有很多相应的讨论。

如果说v2/v3只是"用react组件声明路由配置"的话，v4已经抱着"路由即组件"的方针把耦合侵入到了渲染阶段。从官网的quick start就能看出来：

```js
// 业务组件
const Topics = ({ match }) => (
  <div>
    <h2>Topics</h2>
    <ul>
      <li>
        <Link to={`${match.url}/rendering`}>
          Rendering with React
        </Link>
      </li>
      <li>
        <Link to={`${match.url}/components`}>
          Components
        </Link>
      </li>
      <li>
        <Link to={`${match.url}/props-v-state`}>
          Props v. State
        </Link>
      </li>
    </ul>
    {/*路由配置*/}
    <Route path={`${match.url}/:topicId`} component={Topic}/>
    <Route exact path={match.url} render={() => (
      <h3>Please select a topic.</h3>
    )}/>
  </div>
)
```

有人认为这种设计"extremely make sense"，然而我们却认为种深度侵入的耦合非常危险——一旦接受了这种耦合，就几乎没有任何替换掉它的可能。

然而，抛开react-router的前提是替换其周边配套，比如`Link`组件、`react-router-redux`等，这需要一定的成本。

因此，我们采取的策略是先降低耦合，目前我们使用了react-router@4，但采用扁平、集中的路由配置：

```js
<Switch>
  <Route exact path="/products/:id/edit" component={Product.EditProduct} />
  <Route exact path="/store-product" component={Product.StoreProduct} />
</Switch>
```

以上代码即使你不看react-router的文档，不知道什么是"路由即组件"，也能够快速理解和维护。

这就是集中化配置的优势，react-router那堆服务于路由即组件的古怪用法在我看来是没有意义的学习成本——何况这个库在版本间API巨大变更上是有前科的。

对于开发者和维护者，使用路由库时关注的事情本应非常简单：

1. 如何声明url到渲染函数的映射关系
2. 如何找到一个url对应的渲染函数

扁平、集中化的配置用更低成本的满足了以上需求，也同时降低了路由对整个系统代码的耦合，使得替换更加容易。

## 菜单与面包屑

菜单和面包屑是常见的导航组件。组件本身没有难度，难点在于其背后依赖的数据结构的定义和管理，它必须满足如下：

1. 树状结构
2. 包含权限信息
3. 包含可选的路由信息

实践中有人尝试基于树状的路由配置，添加权限来构建导航树。然而这样有两个风险：

1. 路由与导航菜单的路径结构并不总能一一对应
2. 导航菜单上有一些父节点是展开按钮，没有对应的路由

因此我们决定在路由配置之外，单独维护一份**面向菜单和导航**的配置，即`src/shared/constants/navigation.js`。

```js
const MenuConfig = {
  id: 'root',
  name: 'Home',
  link: '/',
  icon: 'fa-home',
  children: [
    //...other  configs
    {//作为根结点的直接子元素，即第一级导航
      id: 'example',
      name: 'Example', // 在菜单、面包屑中显示的名称
      link: '/example', //页面的链接，如果是类似/user/:id的页面则配置为 pattern: '/user/:id'
    },
    {
      id: 'example2',
      name: 'Example-2',
      pattern: '/example/:id', // url中含有url param，不能直接访问的，使用pattern而不是link。使用pattern的会被菜单过滤，在面包屑中只显示name而不可点击
    },
  ]
}
```

从作用上说，这份配置更加类似于"站点地图"，它表达了站点的导航关系与权限要求。

另外一种思路是在上述配置中，添加`component: PageComponent`来集成路由配置。理论上确实可行，但这样的混杂反而会导致路由配置不够清晰，最终我们决定接受一小部分的重复，采用两份配置。

> 这一部分没有非常完美的标准答案，必然在重复和灵活之间做出取舍。

# 用户鉴权
## 权限配置
路由的权限配置也由`navigation.js`承担。这是因为实际项目中，父子菜单更容易出现权限继承的情况。

```js
  {
    id: 'example',
    name: 'Example', 
    link: '/example', 
    permissions: ['PERM1', 'PERM2'], //权限配置
    children: [
      {
        id: 'child',
        name: 'Child',
        link: '/example-child',
        permissions: ['PERM1']
      }
    ]
  },
```

此处新增了permissions字段，它的值是一个权限数组，若用户拥有其中的一个权限，则具备访问该路由(或展示该菜单)的权限。

子菜单可以不配置permissions，默认继承父菜单的权限。需要注意的是，如果要给子菜单配置权限，则必须是父菜单权限的**子集**。这其实非常好理解：用户必然是先看见上级菜单，才能进入下一级。因此，子菜单要求某权限意味着父菜单也必然要求同一权限。

## 拦截与跳转 * 

从需求上，权限的拦截大致分三类：

1. 对未登录用户，尝试进入要求登录的页面时，跳转回登录页
2. 对已登录用户，尝试进入登录页时，跳转回*登录用户默认页*或*404页*
3. 对已登录用户，尝试访问自己没有权限的页面时，跳转回*登录用户默认页*或*404页*

上面出现了一个定义，即*登录用户默认页*，这是指用户登录进入系统时默认的页面，它可能是配置的，也可能基于某种约定。在本项目的实现中，约定是*在navigation.js中第一个该用户可访问的页面*。

目前我们的鉴权跳转，也通过高阶组件来实现，核心代码如下：

```js
const connectAuthCheck = checkFn => (Content) => {
  class AuthCheck extends Component {
    constructor() {
      super();
      this.state = {
        authed: false,
      };
    }
    componentWillMount() {
      this.doCheck(this.props);
    }
    doCheck(props) {
      return Promise.resolve(checkFn(props))
        .then(() => {
          this.setState({ authed: true });
        }, (e) => {
          console.warn('Page you requested is not allowed: ', e);
        });
    }
    render() {
      const { authed } = this.state;
      return (
        authed ? <Content {...this.props} /> : null
      );
    }
  }

  return connect()(AuthCheck);
};
```

这样只需要定制不同的`checkFn`，即可实现上述三个鉴权需求，以第一点为例：

```js
const connectCheckLogout = connectAuthCheck(props => userStorage.getUser()
  .then(() => {
    props.dispatch(routerActions.replace(routes.DASHBOARD));
    throw new Error('User has already logged in');
  }, () => {}));
```

再在需要的页面使用这个`connectCheckLogout`高阶组件即可，例如登录页：

```js
const decorator = _.flowRight(
  //...other decorators
  connectCheckLogout,
  //...other decorators
);

export default decorator(Login);
```

以上是实现第一点需求的过程，这个需求最简单，适合理解整体架构。

对于第二、三点需求，由于都是对*已登录*用户的鉴权，实践中通常做到一起，同时还承担了`派发action向store中注入user信息`的职责，因此相对复杂：

```js
const checkLogin = props => userStorage.getUser()
  .then((data) => {
    props.dispatch(accountActions.ensureUser(data)); // 派发action向store中注入user信息
    return data;
  }, (e) => {
    props.dispatch(routerActions.replace(routes.SIGN_IN));
    throw e;
  });

const checkPermission = props => checkLogin(props)
    .then((user) => {
      const isVisible = checkPathVisible(props.location.pathname, user.permissions);

      if (!isVisible) {
        const path = getFirstVisiblePath(user.permissions); 
        props.dispatch(routerActions.replace(path));
        throw new Error(`Please check user's permission.`);
      }
      return user;
    });

const connectCheckPermission = connectAuthCheck(checkPermission);
```

这里确实违反了**单一职责原则**，但这部分代码本身复杂度不高，变化可能也非常低，在这里付出一些dirty，换quick和easy是可以接受的。