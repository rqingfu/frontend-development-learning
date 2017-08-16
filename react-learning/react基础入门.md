# React基础入门

React是用于构建用户交互界面的js库, 围绕React有很丰富的生态。通常React和react-router, redux, webpack, es6等结合起来开发应用。React具有如下特点:

- 描述式的

    React让创建交互界面变得轻松。React是数据驱动的视图，每次(状态)的数据的改变, React都会高效的更新和渲染每个组件。这种数据驱动的方式使代码可预测，容易理解和调试。

- 基于组件的

    React构建封闭的组建来管理其状态，从而构建更复杂的逻辑。React由于使用组件而不是模版，从而可以传递根据丰富的数据格式和不需要把需要维护的状态绑定到dom。此外，也可以实现代码的复用。

- learn once, write anywhere

## 安装
### cnpm
由于npm安装依赖包速度太慢，先安装cnpm:
```
npm install -g cnpm --registry=https://registry.npm.taobao.org
```
或者你直接通过添加 npm 参数 alias 一个新命令:
```
alias cnpm="npm --registry=https://registry.npm.taobao.org --cache=$HOME/.npm/.cache/cnpm --disturl=https://npm.taobao.org/dist --userconfig=$HOME/.cnpmrc"

# Or alias it in .bashrc or .zshrc
$ echo '\n#alias for cnpm\nalias cnpm="npm --registry=https://registry.npm.taobao.org --cache=$HOME/.npm/.cache/cnpm --disturl=https://npm.taobao.org/dist --userconfig=$HOME/.cnpmrc"' >> ~/.zshrc && source ~/.zshrc
```

### 创建一个新的React App:
#### 使用create-react-app构建
```
cnpm install -g create-react-app
create-react-app my-app

cd my-app
npm start
```

#### 手工构建（ES6 babel react react-router webpack）
创建文件夹并使用npm init初始化。
(假设当前路径为～/local-repertory)
```
mkdir react-demo
cd react-demo
npm init
```
react目录结构
(假设当前路径为～/local-repertory/react-dom)
```
- - src
| |- components/
- |- index.jsx
|
- - templates
| |- index.html
-package.json
|
-webpack.config.js
```
编辑package.json文件:
(涉及依赖包及运行脚本部分)
```
{
  "name": "react-demo",
  "version": "1.0.0",
  "description": "react demo",
  "main": "index.js",
  "scripts": {
    "dev": "webpack-dev-server",
    "build": "webpack -p",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [
    "react",
    "es6",
    "webpack",
    "babel"
  ],
  "author": "qingfu.ren",
  "license": "MIT",
  "dependencies": {
    "react": "^15.6.1",
    "react-dom": "^15.6.1",
    "react-router": "^3.0.5"
  },
  "devDependencies": {
    "babel-cli": "^6.24.1",
    "babel-core": "^6.25.0",
    "babel-loader": "^7.1.1",
    "babel-preset-env": "^1.6.0",
    "babel-preset-react": "^6.24.1",
    "css-loader": "^0.28.4",
    "less-loader": "^4.0.5",
    "style-loader": "^0.18.2",
    "url-loader": "^0.5.9",
    "webpack": "^2.7.0",
    "webpack-dev-server": "^2.7.1"
  }
}
```
安装必要的包
```
cnpm install
```
编辑webpack.config.js文件:
```
var path = require('path');

module.exports = {
  entry: path.resolve(__dirname, './src/index.jsx'),
  output: {
    path: path.resolve(__dirname, './build'),
    filename: 'bundle.js',
  },
  module: {
    rules: [{
      test: /\.jsx?$/,
      exclude: /(node_modules|bower_components)/,
      use: {
        loader: 'babel-loader',
        options: {
          presets: ['env', 'react']
        }
      }
    }, {
      test: /\.css$/,
      exclude: /(node_modules|bower_components)/,
      use: ['style-loader', 'css-loader']
    }, {
      test: /\.less$/,
      exclude: /(node_modules|bower_components)/,
      use: ['style-loader', 'css-loader', 'less-loader']
    }, {
      test: /\.(png|jpg)$/,
      exclude: /(node_modules|bower_components)/,
      use: ['url-loader?limit=25000']
    }]
  },
  devServer: {
    port: 7001,
  }
};
```
html模版:
```
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>React Demo</title>
</head>
<body>
  <div id="root"></div>
  <script src="/bundle.js"></script>
</body>
</html>
```

## example: Hello world
示例代码:
```
import React, {Component} from 'react';

class HelloWorld extends Component {
  render() {
    return (<div>hello, world</div>);
  }
}

export default HelloWorld;
```
向模版中root元素写入\<div>Hello, world\</div>。这里涉及到react的特殊语法:\<div>hello, world\</div>, 即好像在js中写html。下面讲一下JSX。

## JSX
JSX本质上既不是字符串也不是html，是对js语法的一个扩展。React推荐使用JSX来描述UI的长相，JSX和模版语言也不同，它可以充分的利用js能力。jsx本质上产生React“元素”，然后在该“元素”映射到展示的DOM元素上（React完成，无需我们惯性）。
关于JSX需要掌握的一些知识点:
### JSX中内嵌表达式
JSX允许在其中嵌入js表达式，该表达需要在大括号({})中。如：
```
function formatName(user) {
  return user.firstName + ' ' + user.lastName;
}

const user = {
  firstName: 'Harper',
  lastName: 'Perez'
};

const element = (
  <h1>
    Hello, {formatName(user)}!
  </h1>
);

ReactDOM.render(
  element,
  document.getElementById('root')
);
```
通常，为了使JSX具有良好的阅读下性，建议将jsx写到多行中（非必须）；采用多行时，需要使用括号来包裹该JSX代码，避免js自动在jsx中插入分号。
### JSX自身也是表达式
JSX在js中可以当作表达式使用，如下：
```
function getGreeting(user) {
  if (user) {
    return <h1>Hello, {formatName(user)}!</h1>;
  }
  return <h1>Hello, Stranger.</h1>;
}
```
### JSX 属性值规范
在JSX中有两种方式添加属性值，如：
```
(1) const element = <div tabIndex="0"></div>;
(2) const element = <img src={user.avatarUrl}></img>;
```
（1）中的tabIndex属性使用的字符串作为属性值；（2）中的src使用的时js中表达式的结果作为属性值。需要注意使用表达式时需要把js表达式放入大括号（{}）中，同时不需要像字符串那样使用引号（通常推荐使用双引号）。
### JSX中子节点规范
空元素使用/>结尾，如：
```
const element = <img src={user.avatarUrl} />;
```
对于有子节点的元素，如下：
```
const element = (
  <div>
    <h1>Hello!</h1>
    <h2>Good to see you here.</h2>
  </div>
);
```
需要注意，在jsx中采用驼峰式（camelCase）命名，对应的DOM property也如此，如html的tabindex在html中命名为tagIndex。此外html的class对应为jsx中的className。
### JSX可以阻止注入式攻击（免疫XXS攻击）
JSX对如下形式的输入是安全的：
```
const title = response.potentiallyMaliciousInput;
// This is safe:
const element = <h1>{title}</h1>;
```
如果title中的内容是html字符串，JSX会进行转译，从而确保安全，是网站避免遭受XXS攻击。
### JSX的本质
Babel会将JSX转化为React.createElement()调用。如下两段代码是等价的：
```
const element = (
  <h1 className="greeting">
    Hello, world!
  </h1>
);
```
```
const element = React.createElement(
  'h1',
  {className: 'greeting'},
  'Hello, world!'
);
```
React.createElement做了一些校验来帮助你减少不必要的bug，其本质上返回的是如下的对象：
```
// Note: this structure is simplified
const element = {
  type: 'h1',
  props: {
    className: 'greeting',
    children: 'Hello, world'
  }
};
```
上述对象被称作React元素。React通过这些React元素来构建DOM元素并更新DOM元素。
## 渲染React元素
React元素描述了你期望在浏览器中希望看到效果，如：
```
const element = <h1>Hello, world</h1>;
```
和浏览器的DOM元素不同，React元素只是普通的对象，创建的成本低。ReactDOM负责把React元素映射（更新）到HTML DOM元素。
### 渲染React元素到DOM中
在我们搭建的项目的templates目录下的index.html模版中有如下DOM元素：
```
<div id="root"></div>
```
我们把该DOM元素称作ReactDOM管理React元素到HTML DOM的根DOM节点。使用React构建的项目都需要有一个这样的根DOM节点，当我们期望把React整合到已有项目时，我们可以根据需要选择根DOM节点。使用下面的ReactDOM.render方法来把React元素插入进根DOM节点：
```
const element = <h1>Hello, world</h1>;
ReactDOM.render(
  element,
  document.getElementById('root')
);
```
### 更新已渲染的React元素
React元素时不可变的（immutable），即一旦你创建了React元素，你就不能在改变它的子元素和属性。React元素就类似于动画中的一帧：即该React元素代表某特定时刻的UI被传给了ReactDOM.render方法。如下代码：
```
function tick() {
  const element = (
    <div>
      <h1>Hello, world!</h1>
      <h2>It is {new Date().toLocaleTimeString()}.</h2>
    </div>
  );
  ReactDOM.render(
    element,
    document.getElementById('root')
  );
}

setInterval(tick, 1000);
```
通过setInterval的回调每秒钟执行一次ReactDOM.render来更新React元素。
### React仅仅更新必要部分
上述的更新渲染并没有更新整个Ract元素，而是仅仅更新了改变的部分。
## 组件（Components）和 props
组件本质上是把UI分解为独立的，可复用的代码片段，并且独立的处理各个片段。组件和js函数类似，其接受任意的输入(称作'props')并返回React元素。
### 函数式（Functional）和类式（class）组件
函数式组件形式如下：
```
function Welcome(props) {
  return <h1>Hello, {props.name}</h1>;
}
```
类式组件形式如下：
```
class Welcome extends React.Component {
  render() {
    return <h1>Hello, {this.props.name}</h1>;
  }
}
```
上述两个组件从React的角度开式等价的，但类似组件能比函数式组件提供更多的特性，如下面要涉及的组件state和生命周期方法。
### 渲染组件
```
function Welcome(props) {
  return <h1>Hello, {props.name}</h1>;
}

const element = <Welcome name="Sara" />;
ReactDOM.render(
  element,
  document.getElementById('root')
);
```
### prop是只读的, 不能修改
React足够灵活但有一条严格的规则：
**所以React组件必须相对于它们的props看起来像纯函数。**
## state & 生命周期方法
### state
state用来维护组件的状态，同时随着state的变化来更新组件相应的UI。可以将组件看成是一个状态机，刚开有一个初始状态，随着交互或时间的推移，组件state在变化，和state相关的UI也自动更新。
#### state的正确使用姿势
- 不要直接修改state
错误方式：
```
this.state.comment = 'Hello';
```
正确方式：
```
this.setState({comment: 'Hello'});
```
唯一可以赋值this.state的地方时构造函数内。
- state的更新是异步的
React处于性能考虑可能会将多个setState更新合并为一个更新。因此下面的更新不正确：
```
this.setState({
  counter: this.state.counter + this.props.increment,
});
```
正确的更新方式：
```
this.setState((prevState, props) => ({
  counter: prevState.counter + props.increment
}));
// 或者
this.setState(function(prevState, props) {
  return {
    counter: prevState.counter + props.increment
  };
});
```
- state以合并的方式更新
当调用setState时，React会把你提供的对象合并到当前的state中，但这中合并时浅层次的合并。
### 生命周期方法
React为组件提供了如下生命周期方法：
当组件实例（React元素）被创建并插入DOM根节点时，如下生命周期方法将被调用：
```
constructor()
componentWillMount(object nextProps, object nextState)
render()
componentDidMount(object prevProps, object prevState)
```
当React元素更新时，如下生命周期方法将被调用：
```
componentWillReceiveProps(object nextProps)
shouldComponentUpdate(object nextProps, object nextState)
componentWillUpdate(object nextProps, object nextState)
render()
componentDidUpdate(object prevProps, object prevState)
```
当React元素卸载时，如下生命周期方法将被调用：
```
componentWillUnmount()
```
### example
考虑上述【更新已渲染的React元素】小节中的例子：
```
function tick() {
  const element = (
    <div>
      <h1>Hello, world!</h1>
      <h2>It is {new Date().toLocaleTimeString()}.</h2>
    </div>
  );

  ReactDOM.render(
    element,
    document.getElementById('root')
  );
}

setInterval(tick, 1000);
```
可以考虑据此做一个Clock组件，如：
```
function Clock(props) {
  return (
    <div>
      <h1>Hello, world!</h1>
      <h2>It is {props.date.toLocaleTimeString()}.</h2>
    </div>
  );
}

function tick() {
  ReactDOM.render(
    <Clock date={new Date()} />,
    document.getElementById('root')
  );
}

setInterval(tick, 1000);
```
但这样做却忽略了一个重要的事实：设置timer以及每秒更新UI是Clock应该关心的事实，应该在Clock组件内实现。因此标准的实现方式需要借助State和生命周期函数。如下：
```
class Clock extends React.Component {
  constructor(props) {
    super(props);
    this.state = {date: new Date()};
  }

  componentDidMount() {
    this.timerID = setInterval(
      () => this.tick(),
      1000
    );
  }

  componentWillUnmount() {
    clearInterval(this.timerID);
  }

  tick() {
    this.setState({
      date: new Date()
    });
  }

  render() {
    return (
      <div>
        <h1>Hello, world!</h1>
        <h2>It is {this.state.date.toLocaleTimeString()}.</h2>
      </div>
    );
  }
}

ReactDOM.render(
  <Clock />,
  document.getElementById('root')
);
```
## 事件处理
React元素的事件处理和DOM元素的事件处理类似，有如下语法上的不同：
- React事件采用驼峰（camelCase）命名，不是小写的方式；
- React传入的事件处理是函数，不是字符串。
```
// html
<button onclick="activateLasers()">
  Activate Lasers
</button>
// JSX
<button onClick={activateLasers}>
  Activate Lasers
</button>
```
另外，在React中不能通过return false来阻止默认行为，你必须明确的调用preventDefault。
```
class Toggle extends React.Component {
  constructor(props) {
    super(props);
    this.state = {isToggleOn: true};

    // This binding is necessary to make `this` work in the callback
    this.handleClick = this.handleClick.bind(this);
  }

  handleClick() {
    this.setState(prevState => ({
      isToggleOn: !prevState.isToggleOn
    }));
  }

  render() {
    return (
      <button onClick={this.handleClick}>
        {this.state.isToggleOn ? 'ON' : 'OFF'}
      </button>
    );
  }
}

ReactDOM.render(
  <Toggle />,
  document.getElementById('root')
);
```
或者
```
class LoggingButton extends React.Component {
  handleClick() {
    console.log('this is:', this);
  }

  render() {
    // This syntax ensures `this` is bound within handleClick
    return (
      <button onClick={(e) => this.handleClick(e)}>
        Click me
      </button>
    );
  }
}
```
## FORM
React中的form和HTML中的form在用法上存在差异。
### 受控组件
在HTML中，form元素，如\<input>, \<textarea>和\<select>维护它们自己的状态和根据用户的输入作出更新。在React中，可变状态通常保存在组件的state中并只能通过setState更新。通过React来控制其值的form元素被称作“受控组件”。

```
class NameForm extends React.Component {
  constructor(props) {
    super(props);
    this.state = {value: ''};

    this.handleChange = this.handleChange.bind(this);
    this.handleSubmit = this.handleSubmit.bind(this);
  }

  handleChange(event) {
    this.setState({value: event.target.value});
  }

  handleSubmit(event) {
    alert('A name was submitted: ' + this.state.value);
    event.preventDefault();
  }

  render() {
    return (
      <form onSubmit={this.handleSubmit}>
        <label>
          Name:
          <input type="text" value={this.state.value} onChange={this.handleChange} />
        </label>
        <input type="submit" value="Submit" />
      </form>
    );
  }
}
```

## 参考链接
- [React官方文档](https://facebook.github.io/react/docs/hello-world.html)
- [React 入门实例教程](http://www.ruanyifeng.com/blog/2015/03/react.html)
