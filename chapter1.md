# 代码规范

## 运行检查

为了方便开发，我们配置了`precommit`的钩子脚本，在每次`git commit`时前置检查当前staged的文件，并尝试[自动修复](http://eslint.org/docs/user-guide/command-line-interface#--fix)，保证提交的代码符合代码规范。

你可以使用`npm run precommit`单独运行上述前置检查，也可以用`npm run lint`执行全项目检查(不自动修复)。

如果希望在`git commit`时跳过风格检查，请使用[`git commit -n`](https://git-scm.com/docs/git-commit#git-commit--n) (*不推荐*)。

## 规则概述

本项目代码规范基于 [Airbnb JavaScript Style Guide](https://github.com/airbnb/javascript)。

Airbnb在react技术栈有丰富的实践经验，也是社区的活跃贡献组织。该Style guide不仅提供了eslint配置，对每条规则都有详细的代码示例和说明，同时提供了`How`与`Why`，非常适合新人学习。

在Airbnb规则的基础上，我们对以下规则做了扩展和修改：

```js
"rules": {
  "quotes": [
    "error",
    "single",
    {
      /*
      * 允许使用没有插入变量的模板字符串
      * See: http://eslint.org/docs/rules/quotes#allowtemplateliterals
      */
      "allowTemplateLiterals": true 
    }
  ],
  "no-unused-expressions": [
    "error",
    {
      /*
      * 允许使用 a && b() 与 a ? b() : c() 这样的未赋值表达式来控制逻辑
      */
      "allowShortCircuit": true, 
      "allowTernary": true
    }
  ],
  "jsx-a11y/label-has-for": 1, // label不强制要求for属性，报错级别降为warning
  /*
  * 此规则的本意是保障网站的可访问性(Accessibility)
  * 要求div/p等语义上不可交互的元素除非添加role属性否则不能处理交互事件(例如onClick)。
  * 对于可访问性有要求的项目，建议采用默认配置
  * 否则，建议将报错级别降为warning (即本项目采用的配置)
  * See: https://github.com/evcohen/eslint-plugin-jsx-a11y/blob/master/docs/rules/no-static-element-interactions.md
  */
  "jsx-a11y/no-static-element-interactions": 1, 
  /*
  * 此规则禁用嵌套的三元表达式，并建议使用if/else语句代替
  * 然而我们认为三元表达式更加函数式，嵌套的结构相比过程式的if/else更能反映真实的逻辑树。
  */
  "no-nested-ternary": [
    1
  ],
}

```

还有一个规则需要特别注意，即[`react/no-multi-comp`](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/no-multi-comp.md)，该规则限制一个文件只能声明一个组件，这在多数情况下更加清晰，因此我们没有重写这条规则，建议尽量遵守。

然而，在少数场景下，需要创建一些简单的"辅助"组件，最典型的场景是列表：

```js
var List = createReactClass({
  render() {
    return (
      <ul>
        {this.props.items.map(item =>
          <li key={item.id} onClick={this.props.onItemClick.bind(null, item.id)}>
            ...
          </li>
        )}
      </ul>
    );
  }
});
```

此处我们使用了bind为事件处理函数绑定数据。bind意味着创建一个新的函数，特别是在列表中，每次render都会多次调用bind。而react从机制上是有可能频繁触发render函数的，这对性能有相当明显的负面影响。

对此，我们有[`jsx-no-bind`](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/jsx-no-bind.md)规则，禁止在render中使用bind。

可是禁用了bind，怎么为列表的子元素绑定数据呢？目前社区比较认可的做法是创建一个子组件：

```js
var List = createReactClass({
  render() {
    return (
      <ul>
        {this.props.items.map(item =>
          <ListItem key={item.id} item={item} onItemClick={this.props.onItemClick} />
        )}
      </ul>
    );
  }
});

var ListItem = createReactClass({
  render() {
    return (
      <li onClick={this._onClick}>
        ...
      </li>
    );
  },
  _onClick() {
    this.props.onItemClick(this.props.item.id);
  }
});
```

ListItem组件没有任何业务上的意义，仅仅是为了取代bind而存在，另起一个文件显得非常多余。但以上代码又显然违反了`react/no-multi-comp`(一个文件只能声明一个组件)规则。

对这类少数情况，我们建议在文件顶部加上`/* eslint-disable react/no-multi-comp */`，暂时屏蔽`react/no-multi-comp`。

不止`react/no-multi-comp`，有限的、死板的规则注定无法应对灵活的业务场景。如果一条规则对当前的开发造成了阻碍，请思考这条规则是否属于"多数场景有必要遵守"，如果不是，可以考虑修改全局的`.eslintrc`，如果是，则考虑在[局部disable](http://eslint.org/docs/user-guide/configuring#disabling-rules-with-inline-comments)。在code review中，也最好对eslint规则的改动、禁用重点关注，以保证代码质量。