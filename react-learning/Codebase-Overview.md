# Codebase Overview

这部分将给出React代码库组织，规范和实现的概览。

如果你想要对React做贡献我们希望该指南会帮你更自信的做修改。

我们并不会在React应用中推荐这些规范。一些规定的存在是由于历史原因并可能随时间改变。

## 自定义模块系统

在Facebook内部，我们使用叫做Haste的自定义模块系统。其和CommonJS类似，也使用require，但是有一些重要的不同的点。这些点经常会使外部的贡献者感到迷惑。

在commonJS中，当你引入模块时，你需要具体指出其相对路径：

```javascript
// Importing from the same folder:
var setInnerHTML = require('./setInnerHTML');

// Importing from a different folder:
var setInnerHTML = require('../utils/setInnerHTML');

// Importing from a deeply nested folder:
var setInnerHTML = require('../client/utils/setInnerHTML');
```

然而，在Haste中所有文件名是全局唯一的。在React代码库中，你可以只通过模块名从任何模块引入其他模块：

```javascript
var setInnerHTML = require('setInnerHTML');
```

Haste起初被开发来给像Facebook这样的大型应用。其很容易的把文件迁移到不同的文件夹并且引用它们不需要担心相对路径。在任何编辑器中模糊查询文件总能指引到正确的地方，因为是全局唯一的名字。

React本身是从Facebook代码库中抽取出来的，使用Haste是历史原因。在未来，我们可能会将React迁移为使用CommonJS和ES模块，目的是和社区看齐。然而，这要求在Facebook内部基础平台的改变，因此其不可能马上发生改变。

<strong>如果你记住如下规则Haste会对你更加有意义：</strong>

- 所有在React源码中的文件名都是全局唯一的。这也是为什么有时文件名会比较冗长。
- 当你添加新文件时，确保你包含一个协议头。你可以从任何存在的文件中拷贝。协议头中总是包含这样一行。修改它来和你创建的文件名匹配。
- 当引用模块时不要使用相对路径。不是，而是写。


当我们使用npm编译React时，脚本会拷贝所有模块到一个叫lib的文件夹并在所有require路径前添加./。通过这种方式，Node，Browserify，Webpack和其他工具可以在不知Haste的情况下知道React构建输出。

如果你正在阅读GitHub上的React源码且想要跳到某个文件，按下"t"。

这是GiyHub在当前代码库中模糊匹配文件名的快捷键。输入你要查找的文件名，然后其会作为第一条匹配出现。

## 外部依赖

React几乎没有外部依赖。通常，require指向React代码库中的文件。然而，有一些罕见的列外。

如果你看到Require不是对应着React代码库中的文件，你可以在一个叫做fbjs的特殊库中查找。比如，warning将解析为来着fbjs模块的warning。

fbjs库的存在是因为React会和像Relay这样的库共享一下小的实用模块，我们确保它们是同步的。我们不依赖node生态系统中等价的小模块是因为我们想要Facebook工程师在必要的时候可以做修改。在fbjs中的实用模块并不认为是对外公开的API，它们仅仅是为了让像React这样的Facebook项目使用。

## 顶级文件夹

当克隆React库后，你可以看到库中的如下顶级文件夹：

- src是React源码。如果你只是修改源码，src是你花费时间最多的地方。
- docs是React文档网站。当你改变APIs时，确保及时更新相关Markdown文件。
- packages包含在React库中所有包的元数据（如package.json）。然而，它们的源代码仍然位于src中。
- build是react的构建输出。它并不在库中，但是当你第一次构建它后其会出现在你的克隆库中。


还有其他一些顶级文件夹，但是它们大多是被当作工具使用。当你对代码做贡献时你可能不会遇到它们。

## 测试位置

我们没有针对单元测试的顶级目录。相反，我们把单元测试放到__test__目录中，该目录和要测试的文件在同一层。

比如，对setInnerHTML.js的测试在紧挨在其的__test__/setInnerHTML-test.js中。

## 共用代码

尽管Haste允许我们从库任何地方引入任何模块，我们会遵守一些规范来避免循环依赖和其他令人不快的惊喜。通过规约，文件可能只能引用同一目录或子目录中的文件。

比如，在src/renderers/dom/stack/client的文件可以引用同目录的其他文件或同目录下的任何目录。

然而它们不能引用来自src/renderer/dom/stack/sever的模块，因为其不是src/renderers/dom/stack/client目录的子目录。

这条规则有例外。有时我们确实需要在两个模块组中公用功能。在这种情况下，我们提升共用的模块到一个叫shared的文件夹，该文件夹位于需要使用该共享模块到模块的同一路径中的相同的文件中。

比如，src/renderers/dom/stack/client和src/renderers/dom/stack/server的共享模块位于src/renderers/dom/shared中。

同样的逻辑，如果src/renderers/dom/stack/client需要和src/renderers/native共享一些模块，该共享模块位于src/renderers/shared中。

该规范并不是强制的但是我们会在pull request审核时检查它。

## 警告和不变量

React库使用warning模块来展示警告信息：
```javascript
var warning = require('warning');

warning(
  2 + 2 === 4,
  'Math is not working today.'
);
```

当warning条件为false时warning信息展示。

考虑的一种方式时条件应该反应正常情况而不是一个异常。

避免重复的输出警告是一个不错的主意：

```javascript
var warning = require('warning');

var didWarnAboutMath = false;
if (!didWarnAboutMath) {
  warning(
    2 + 2 === 4,
    'Math is not working today.'
  );
  didWarnAboutMath = true;
}
```

警告仅仅在开发模式开启。在生产模式，它们完全被关闭。如果你需要禁止某些代码执行，使用invariant模块代替：

```javascript
var invariant = require('invariant');

invariant(
  2 + 2 === 4,
  'You shall not pass!'
);
```

当不变量条件为false时不变量被抛出。

"不变量"仅仅是说该条件总是true的一种方式。你可以认为它是一种断言。

保持开发和生产模式中行为的相似是重要的，因此invariant在开发和生产环节中都会被抛出。错误信息在生产模式中被自动替换成错误码来避免字节大小的负面影响。

## 开发和生产

你可以在代码库中使用__DEV__伪全局变量来标示仅仅是开发的代码块。

在编译阶段其被内联，并在CommonJS中转化为process.env.NODE_ENV ！==production。检查。

对于单独构建，在没有压缩的构建中它是true，在压缩构建中在if代码块中的代码被完全剔除。

```javascript
if (__DEV__) {
  // This code will only run in development.
}
```

## JSDoc
一些内部和公共方法被使用JSDoc注释里注释：

```javascript
/**
  * Updates this component by updating the text content.
  *
  * @param {ReactText} nextText The next text content
  * @param {ReactReconcileTransaction} transaction
  * @internal
  */
receiveComponent: function(nextText, transaction) {
  // ...
},
```

我们尽力让存在的注释保持最新当时我们不会强制它们。在新近写的代码中我们不使用JSDoc，使用Flow来文档化和强制类型。

## Flow

我们最近开始使用Flow检查代码库。在协议头标记@flow注解的文件会被检查。

我们接受添加Flow注解到已有代码的拉取请求。Flow注解如下所示：

```javascript
ReactRef.detachRefs = function(
  instance: ReactInstance,
  element: ReactElement | string | number | null | false,
): void {
  // ...
}
```

新代码应该尽可能的使用Flow注解。

你可以在本地允许npm run flow来用flow检查你的代码。

## 类和mixins

React最初使用ES5编写。我们已经使用Babel开启了es6特性，包括类。然而，大多数React代码仍然是使用ES5写的。

特别，你可能经常看到如下模式：

```javascript
// Constructor
function ReactDOMComponent(element) {
  this._currentElement = element;
}

// Methods
ReactDOMComponent.Mixin = {
  mountComponent: function() {
    // ...
  }
};

// Put methods on the prototype
Object.assign(
  ReactDOMComponent.prototype,
  ReactDOMComponent.Mixin
);

module.exports = ReactDOMComponent;
```

在代码中Mixins和React Mixins特性没有关系。其是组合对象一些函数的方式。这些方法可能之后被固定到一些其他类。尽管我们努力在新代码中避免它但我们还是在一些地方使用该模式。

等价的ES6代码像下面这样：

```javascript
class ReactDOMComponent {
  constructor(element) {
    this._currentElement = element;
  }

  mountComponent() {
    // ...
  }
}

module.exports = ReactDOMComponent;
```

有时我们把旧代码转化为ES6类。然而，这对于我们并不重要，因为这在努力使用非面向对象的方法来替代React reconciler实现。

## 动态注入

React在一些模块中使用动态注入。尽管它总是明确的，但不幸的是它阻碍了对代码的理解。其存在的主要原因是React初始时仅支持DOM作为目标。React Native开始forkReact。我们不得不添加动态注入来让React Native覆盖一些行为。

你可能看到如下的声明动态依赖的代码：
```javascript
// Dynamically injected
var textComponentClass = null;

// Relies on dynamically injected value
function createInstanceForText(text) {
  return new textComponentClass(text);
}

var ReactHostComponent = {
  createInstanceForText,

  // Provides an opportunity for dynamic injection
  injection: {
    injectTextComponentClass: function(componentClass) {
      textComponentClass = componentClass;
    },
  },
};

module.exports = ReactHostComponent;
```

injection字段不没有以特殊地方式被处理。但是通过约定，这意味着该模块在运行时需要一些品台相关的依赖被注入。

在React DOM中，ReactDefaultInjection注入特定DOM的实现：

```javascript
ReactHostComponent.injection.injectTextComponentClass(ReactDOMTextComponent);
```

在React Native中， ReactNativeDefaultInjection注入它自己的实现：

```javascript
ReactHostComponent.injection.injectTextComponentClass(ReactNativeTextComponent);
```

在代码库中有很多的插入点。未来，我们打算避开动态注入机制，在构建中连接所有的静态片段。

## 多包

React是单一库。它的库包含多个独立的包以便它们的改变可以协调一致，文档和issues在同一个地方。

想pakage.json这样的npm元数据文件在顶级package目录中。然而，在其中并没有真正的代码。

比如，packages/react.js再次导出react.js，其是npm正真的入口。其他包大多重复该模式。所有关键的代码都在src。

然而代码被分割成不同的部分，真正的包边界和npm包略微不同并独立的浏览器构建。

## React核心

React的核心包括所有react对象的顶级API，比如：
- React.createElement()
- React.Component
- React.Children

React核心仅仅包括定义组件的必要API。其不包含reconciliation算法或平台相关代码。这些代码被React DOM和React Native组件使用。

React核心代码在源码树中的isomorphic目录。其可以在npm上作为react包使用。对应的浏览器构建是react.js，其暴露一个全局变量React。
```
注意：
直到最近，react npm包和react.js包含了所有的代码而不是react核心。这样做主要是向后兼容和历史原因。自从15.4.0版本，核心代码在构建输出中是单独的。
```

也有其他额外的单独构建react-with-addons.js，在将来会考虑独立出来。

## 渲染器(Renderers)

React当初因为DOM而被创建，但是之后其也适用支持React Native的本地平台。这里介绍React内部渲染器的概念。

渲染器管理React树如何转变成平台下面的调用。

渲染器在文件夹src/renderers中:

- React DOM Renderer renders React components to the DOM. It implements top-level ReactDOM APIs and is available as react-dom npm package. It can also be used as standalone browser bundle called react-dom.js that exports a ReactDOM global.
- React Native Renderer renders React components to native views. It is used internally by React Native via react-native-renderer npm package. In the future a copy of it may get checked into the React Native repository so that React Native can update React at its own pace.
- React Test Renderer renders React components to JSON trees. It is used by the Snapshot Testing feature of Jest and is available as react-test-renderer npm package.
The only other officially supported renderer is react-art. To avoid accidentally breaking it as we make changes to React, we checked it in as src/renderers/art and run its test suite. Nevertheless, its GitHub repository still acts as the source of truth.

While it is technically possible to create custom React renderers, this is currently not officially supported. There is no stable public contract for custom renderers yet which is another reason why we keep them all in a single place.

```
Note:
Technically the native renderer is a very thin layer that teaches React to interact with React Native implementation. The real platform-specific code managing the native views lives in the React Native repository together with its components.
```

## (调节器)Reconcilers

即使像ReactDOM和React Native这样差异巨大的渲染器需要共享很多逻辑。特别地，reconciliation算法应该尽可能的相似以便在跨平台时refs，生命周期，state，自定义组件和声明式渲染工作一致。

为了解决这个问题，不同的渲染器在它们之间共享代码。我们称这是React reconciler的一部分。当像setState这样的更新函数被调用时，reconciler调用组件上的render函数。

Reconciler不是分开的包因为当前它们没有公开的API。它们被像React DOM和React Native这样的渲染器单独使用。

## 堆栈调节器(Stack Reconciler)

堆栈调节器是React所有生产代码的动力之一。其在src/renderers/shared/stack/reconciler文件夹中并被React DOM和React Native两者使用。

它被以面向对象的方式编写并且维护所有React组件内部实例的独立树。内部实例包括用户定义和特定平台组件。内部实例对用户不能直接使用，且它们的树不对外暴露。

当组件挂载，更新和卸载时，堆栈调和器调用内部实例的方法。这些函数是mountComponent(element), receiveComponent(nextElement), 和unmountComponent(element)。

### 宿主组件

平台相关("宿主")组件，如&lt;div>或&lt;View>运行平台相关代码。比如，React DOM命令堆栈调节器来使用ReactDOMComponent处理DOM组件的挂载，更新和卸载。

不管啥平台，&lt;div>和&lt;View>以相似的方式处理管理很多子组件。为了方便，堆栈调节器提供叫ReactMultiChild的助手，其被DOM和Native渲染器使用。

### 复合组件(Composite Components)
User-defined ("composite") components should behave the same way with all renderers. This is why the stack reconciler provides a shared implementation in ReactCompositeComponent. It is always the same regardless of the renderer.

Composite components also implement mounting, updating, and unmounting. However, unlike host components, ReactCompositeComponent needs to behave differently depending on the user's code. This is why it calls methods, such as render() and componentDidMount(), on the user-supplied class.

During an update, ReactCompositeComponent checks whether the render() output has a different type or key than the last time. If neither type nor key has changed, it delegates the update to the existing child internal instance. Otherwise, it unmounts the old child instance and mounts a new one. This is described in the reconciliation algorithm.

### 递归

在更新期间，堆栈调节器通过复合组件"向下钻"，运行它们的render()函数，且决定是否更新或代替它们的单一孩子实例。当其传入像&lt;div>和&lt;View>这样的宿主组件时其执行平台相关代码。宿主组件可能用多个子元素，它们也会被递归的处理。

了解这点很重要：堆栈调节器总是单一同步地处理组件树。当个别树分支可以脱离reconciliation的同时，堆栈调节器不会暂停，因此当更新很深是是不达标的且可用CPU时间是有限的。

### 了解更多

下一节描述更多当前实现细节。

## 纤维调节器(Fiber Reconciler)

"纤维"调节器是解决堆栈调节器内在问题的新的努力目标且修复了一些长期存在的问题。

它是调和器的完全重写且目前在积极开发中。

它的主要目标是：

- Ability to split interruptible work in chunks.
- Ability to prioritize, rebase and reuse work in progress.
- Ability to yield back and forth between parents and children to support layout in React.
- 能让render()函数返回多个元素。
- 对错误边界更好的支持。

你可以阅读更多关于[React Fiber架构](https://github.com/acdlite/react-fiber-architecture)。现在，其仍然是实验性的，并且和堆栈调和器同等的特性还相去胜远。

其源代码在<strong>src/renderers/shared/fiber中</strong>。

## Event System

React实现了一个合成的事件系统，该系统对渲染器是透明的并且能在React DOM和React Native上运行。这部分代码在<strong>src/renderers/shared/shared/event</strong>文件夹中。

这里有深入钻研其的[视频(66 分钟)](https://www.youtube.com/watch?v=dRo_egw7tBc)。
