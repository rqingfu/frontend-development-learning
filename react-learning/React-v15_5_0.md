
# React v15.5.0

April 7, 2017 by Andrew Clark

从React上一次重大改变到现在已经确切地一年了。我们的下一个大版本，React16，将会包括一些激动人心的改善，这其中就包含React内部的完全重写。我们认认真真地考虑稳定性，并且致力于使用者使用最少的努力来获得这些改善。

为此，今天我们发布了React15.5.0。

## 新弃用警告

最大的改变是我们抽取出React.PropTypes和React.crwateClass形成其独自的包。这两个部分仍然可以通过主React对象访问的，但是在开发模式中使用任意一个将会在console控制台输出一次弃用警告日志。这样做是为了启用对将来代码大小的优化。

这些警告不会对你的应用产生任何的影响。然而，我们认为它们可能会造成一些问题，特别是如果你使用把console.error当作失败的测试框架时。

添加新警告并不是我们能轻易做到的事。在React中警告不仅仅是建议--它们是在最新的React版本中保持尽可能多的人的策略必不可少的。没有提供增量路径我们从不添加警告。

因此尽管在短期内警告会引发问题，我们相信现在催促开发者迁移他们的代码库可以阻止将来更大的问题。主动地修复这些bug确保你为下一个大版本做准备。如果你的应用在15.5版本中的警告数是0，那么在16中不需要任何改变仍然可以运行。

对于每个新的弃用警告，我们已经提供了codemod来自动化地迁移你的代码。它们作为react-codemod项目的一部分被使用。

## 从React.PropTypes中迁移

prop types是在开发期间运行时校验props类型的一个特性。我们抽取内建的proptypes到一个独立的包中来反应这样一个事实：不是所有人都使用它们。

在15.5中，不是从主React对象中访问PropTypes，需安装prop-types包然后从包中引入它们：
```javascript
// Before (15.4 and below)
import React from 'react';

class Component extends React.Component {
  render() {
    return <div>{this.props.text}</div>;
  }
}

Component.propTypes = {
  text: React.PropTypes.string.isRequired,
}

// After (15.5)
import React from 'react';
import PropTypes from 'prop-types';

class Component extends React.Component {
  render() {
    return <div>{this.props.text}</div>;
  }
}

Component.propTypes = {
  text: PropTypes.string.isRequired,
};
```

codemod可以对该变化做自动地转化。基本用法：

```
jscodeshift -t react-codemod/transforms/React-PropTypes-to-prop-types.js <path>
```

propTypes，contextTypes和childContextTypes API和以前的用法一样。唯一的改变是内建的校验器现在是一个单独的包。

你也可以考虑使用Flow来检查你js代码的静态类型，包括React组件。

## 从React.createClass迁移

在React刚被发布的时候，js中还没有符合语言习惯的创建class的方式，因此我们提供了React.createClass。

之后，class语法被添加成为了语言es2015的一部分，因此我们添加了使用js class创建React组件的能力。除了函数式组件，jsclass现在是react中创建组件的首选方式。

对于你代码中存在的createClass组件，我们就推荐你转换它们到js class组件。然而，如果你有组件依赖了mixins，转换为class组件可能不能很快可用。如果真是这样，npm上的create-react-class作为一种降级的替换是可用的：

```javascript
// Before (15.4 and below)
var React = require('react');

var Component = React.createClass({
  mixins: [MixinA],
  render() {
    return <Child />;
  }
});

// After (15.5)
var React = require('react');
var createReactClass = require('create-react-class');

var Component = createReactClass({
  mixins: [MixinA],
  render() {
    return <Child />;
  }
});
```

你的组件将和之前一样可以继续运行。

对于该变化codemod尝试将createClass组件转换为js class组件，如果必要回倒退使用create-react-class。在facebook内部codemod已经转化了上千的组件了。

基本用法：
```
jscodeshift -t react-codemod/transforms/class.js path/to/components
```

## 终止对React Addons的支持

我们将终止积极地维护React Addons包。事实上，这些包中的大多数已经很长时间没有积极地维护了。它们会无限期的运行，但是我们建议尽快迁移出以防止未来被丢弃。

- react-addons-create-fragment – React 16将使用first-class对fragment支持，在这个点上包不会是必要的。我们推荐使用带key的元素数组代替。

- react-addons-css-transition-group - 使用react-transition-group/CSSTransitionGroup。1.1.1版本提供了一个降级替代。

- react-addons-linked-state-mixin - 明确地设置value和onChange处理函数。

- react-addons-pure-render-mixin - 使用React.PureComponent代替。

- react-addons-shallow-compare - 使用React.PureComponet代替。

- react-addons-transition-group - 使用React-transition-group/TransitionGroup代替。

- react-addons-update - 使用immutability-helper代替。

- react-linked-input - 明确地设置value和onChange处理函数。

我们也将终止对UMD构建的react-with-addons的支持。在React 16中将被删除。

## React Test Utils

目前，React Test Utils在react-addons-test-utils中。在15.5版本中，我们会丢弃该包并且把它们移动到react-dom/test-utils中：

```javascript
// Before (15.4 and below)
import TestUtils from 'react-addons-test-utils';

// After (15.5)
import TestUtils from 'react-dom/test-utils';
```

这反应了这样一个事实：调用Test Utils的程序是包裹DOM渲染函数的一组API。

特例是浅渲染，其不是特定的DOM。前renderer已经被移到了react-test-render/shallow中。
```javascript
// Before (15.4 and below)
import { createRenderer } from 'react-addons-test-utils';

// After (15.5)
import { createRenderer } from 'react-test-renderer/shallow';
```

## 致谢

- Jason Miller

- Aaron Ackerman

- Vinicius Marson

## 安装

我们推荐使用Yarn或者npm来管理前端依赖。如果你是新的包管理者，Yarn文档是开始的学习的不错地方。

使用Yarn安装React，运行：

```
yarn add react@^15.5.0 react-dom@^15.5.0
```

使用npm安装React，运行：
```
npm install --save react@^15.5.0 react-dom@^15.5.0
```

我们推荐使用像webpack或Browserify这样的打包器以便你能写模块代码和一起打包进小包中来优化加载时间。

请记住，在默认情况下，开发模式中React运行额外的检查并提供用帮助的警告。当部署应用时，确保是在生产模式中编译。


以防你不使用绑定者，我们也在npm包中提供了预构建的bundle，你可以在你的页面中通过script标签包括它：

具有警告的开发构建
为生产环节的压缩构建

- React <br>
Dev build with warnings: react/dist/react.js <br>
Minified build for production: react/dist/react.min.js

- React with Add-Ons <br>
具有警告的开发构建: react/dist/react-with-addons.js <br>
为生产环节的压缩构建: react/dist/react-with-addons.min.js

- React DOM (include React in the page before React DOM) <br>
具有警告的开发构建: react-dom/dist/react-dom.js <br>
为生产环节的压缩构建: react-dom/dist/react-dom.min.js

- React DOM Server (include React in the page before React DOM Server) <br>
具有警告的开发构建: react-dom/dist/react-dom-server.js <br>
为生产环节的压缩构建: react-dom/dist/react-dom-server.min.js

我们也已经在npm上发布了15.5版本的react、react-dom和addons包和在bower上的react包。
