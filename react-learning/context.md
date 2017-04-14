# react [context V15.5](https://facebook.github.io/react/docs/context.html)章节--翻译

在使用react时，通过React组件追踪数据流是容易的。当你查看组件时，你可以知道那些props被传入，这可以方便的让你推断你的app。
某些情况下，你想要在不通过手工一层层传递props的情况下在组件树中传递数据。在React中你可以使用强大的"context"api来直接这样做。

## 为什么要用context

绝大部分应用（applications）不需要使用context。

如果你想要你的应用稳健，不要使用context。context是实验性的API并且可能在将来发布的react版本中弃用。

假如你不熟悉像Redux或MobX这样的状态管理库，不要使用context。对于多数真实应用，这些库及其React绑定对于管理和多组件相关的state是不错的选择。Redux更可能是对你问题的更好解决方案，与使用context方案相比。

若果你不是一个经验丰富的React开发者，不要使用context。仅仅使用props和state来实现功能对你而言通常是一种更好的方式。

如果在上面这些警告下你仍然坚持要使用context，请尽量对context的使用隔离在小范围内，并且尽可能的避免直接使用context api。这样做的目的是当api改变时可以相对容易的升级。

## 如何使用context
假设你有如下的代码结构：
```javascript
class Button extends React.Component {
  render() {
    return (
      <button style={{background: this.props.color}}>
        {this.props.children}
      </button>
    );
  }
}

class Message extends React.Component {
  render() {
    return (
      <div>
        {this.props.text} <Button color={this.props.color}>Delete</Button>
      </div>
    );
  }
}

class MessageList extends React.Component {
  render() {
    const color = "purple";
    const children = this.props.messages.map((message) =>
      <Message text={message.text} color={color} />
    );
    return <div>{children}</div>;
  }
}
```
在这个例子中，为了适当地给Button和Message组件设置样式我们手动的层层传递prop color。使用context，我们可以自动地通过组件树传递prop color:
```javascript
const PropTypes = require('prop-types');

class Button extends React.Component {
  render() {
    return (
      <button style={{background: this.context.color}}>
        {this.props.children}
      </button>
    );
  }
}

Button.contextTypes = {
  color: PropTypes.string
};

class Message extends React.Component {
  render() {
    return (
      <div>
        {this.props.text} <Button>Delete</Button>
      </div>
    );
  }
}

class MessageList extends React.Component {
  getChildContext() {
    return {color: "purple"};
  }

  render() {
    const children = this.props.messages.map((message) =>
      <Message text={message.text} />
    );
    return <div>{children}</div>;
  }
}

MessageList.childContextTypes = {
  color: PropTypes.string
};
```
通过添加childContextType和getChildContext到MessageList组件（context提供者），React自动地往下传到信息并且在子组件树中的任何组件（在例子中的Button）可以通过定义contextType来访问这些信息。

如果contextType没有定义，那么context将会是一个空对象。

### 父-子连接
context也可以让你构建api来使父组件和子组件通信。比如，React Router V4就是一个以这种机制工作的库：
```javascript
import { BrowserRouter as Router, Route, Link } from 'react-router-dom';

const BasicExample = () => (
  <Router>
    <div>
      <ul>
        <li><Link to="/">Home</Link></li>
        <li><Link to="/about">About</Link></li>
        <li><Link to="/topics">Topics</Link></li>
      </ul>

      <hr />

      <Route exact path="/" component={Home} />
      <Route path="/about" component={About} />
      <Route path="/topics" component={Topics} />
    </div>
  </Router>
);
```

通过从Router组件往下传递一些信息，Link和Route组件可以给父组件Router反馈通信。

在你使用类似上面的api构建组件前，考虑是否能有明确的替代方案。例如，如果你愿意，你可以把整个React组件作为props来传递。

在生命周期方法中引用context
如果在某个组件中定义了context，如下生命周期方法将会接受一个额外的参数，即context对象：
- constructor(props, context)
- componentWillReceiveProps(nextProps, nextContext)
- shouldComponentUpdate(nextProps, nextState, nextContext)
- componentWillUpdate(nextProps, nextState, nextContext)
- componentDidUpdate(prevProps, prevState, prevContext)

在无状态的函数组件中引用context
如果contextType作为函数的property被定义，那么无状态函数也是可以引用context。如下代码展示的是无状态函数编写的Button组件。

```javascript
const PropTypes = require('prop-types');

const Button = ({children}, context) =>
  <button style={{background: context.color}}>
    {children}
  </button>;

Button.contextTypes = {color: PropTypes.string};
```
## 更新Context
请不要这样做。

React有更新context的api，但是其基本上是不好使的，你不应该使用该api。

当state或props改变时getChildContext函数会被调用。为了更新context中的数据，使用this.setState来触发更新本地state更新。这会导致一个新的context并且改变会被子组件接收到。

```javascript
const PropTypes = require('prop-types');

class MediaQuery extends React.Component {
  constructor(props) {
    super(props);
    this.state = {type:'desktop'};
  }

  getChildContext() {
    return {type: this.state.type};
  }

  componentDidMount() {
    const checkMediaQuery = () => {
      const type = window.matchMedia("(min-width: 1025px)").matches ? 'desktop' : 'mobile';
      if (type !== this.state.type) {
        this.setState({type});
      }
    };

    window.addEventListener('resize', checkMediaQuery);
    checkMediaQuery();
  }

  render() {
    return this.props.children;
  }
}

MediaQuery.childContextTypes = {
  type: PropTypes.string
};
```
如果组件提供的context值改变， 将会遇到这样一个问题：如果中间的父组件的shouldComponentUpdate返回false，那么其后的子组件的context不会更新。这完全超出了使用context的组件的控制，因此基本上没有方法来可靠的更新context。这里有一篇博客很好的解释了为什么这是一个问题，并且你怎么可能遇到这个问题。
