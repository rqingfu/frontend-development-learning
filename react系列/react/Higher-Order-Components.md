# Higher-Order Components章节--翻译

高阶组件（HOC）是React中重复利用组件的高级技巧。HOC本身并不是React api的一部分。其是一种模式，出自React构建本性（compositional nature）。

具体来说，高阶函数是一个函数，该函数接收一个组件为参数并返回一个新组件。
```javascript
const EnhancedComponent = higherOrderComponent(WrappedComponent);
```
如同组件把props变为UI，高阶组件把组件转变为另一个组件。

在第三方React库中HOC很常见，如Redux的connect和Relay的createContainer。

在这篇文章当中，我们将讨论高阶组件为什么是有用的，并且如何写自己的高阶组件。

## 对Cross-Cutting Concerns使用HOC

>注意
><br><br>
>我们之前推荐使用mixins作为处理Cross-Cutting Concerns的方式。我们已经意识到mixins带来的麻烦远远大于其带来的价值。可点击这里查看为什么我们抛弃了mixins和怎样对已经存在的组件做过渡。

在React中组件是代码复用的主要单元。然而，你会发现一些模式不能直接适配传统组件。

例如，假如你有一个CommentList组件，该组件订阅一个外部的数据源来渲染一个评论列表：

```javascript
class CommentList extends React.Component {
  constructor() {
    super();
    this.handleChange = this.handleChange.bind(this);
    this.state = {
      // "DataSource" is some global data source
      comments: DataSource.getComments()
    };
  }

  componentDidMount() {
    // Subscribe to changes
    DataSource.addChangeListener(this.handleChange);
  }

  componentWillUnmount() {
    // Clean up listener
    DataSource.removeChangeListener(this.handleChange);
  }

  handleChange() {
    // Update component state whenever the data source changes
    this.setState({
      comments: DataSource.getComments()
    });
  }

  render() {
    return (
      <div>
        {this.state.comments.map((comment) => (
          <Comment comment={comment} key={comment.id} />
        ))}
      </div>
    );
  }
}
```

之后，你写了一个组件来订阅博客，其和上面的模式一样：

```javascript
class BlogPost extends React.Component {
  constructor(props) {
    super(props);
    this.handleChange = this.handleChange.bind(this);
    this.state = {
      blogPost: DataSource.getBlogPost(props.id)
    };
  }

  componentDidMount() {
    DataSource.addChangeListener(this.handleChange);
  }

  componentWillUnmount() {
    DataSource.removeChangeListener(this.handleChange);
  }

  handleChange() {
    this.setState({
      blogPost: DataSource.getBlogPost(this.props.id)
    });
  }

  render() {
    return <TextBlock text={this.state.blogPost} />;
  }
}
```
Comment List和BlogPost并不完全等同--它们调用DataSource上的不同方法，并且它们渲染了不同的输出。但是它们实现的大部分是一样的：

- 挂在时，添加对DataSource的改变监听。
- 在监听函数内部，当数据源改变时调用setState。
- 在卸载时，删除监听函数。

你能想象在大型应用中，订阅DataSource和调用setState的相同模式会一次次的出现。我们需要允许我们在一个地方定义该逻辑的抽象并且在组件中共享它们。这正是高阶组件擅长的地方。

我们可以写一个函数，该函数创建像CommentList和BlogPost的组件去订阅DataSource。函数会接收子组件作为其参数之一，该子组件把订阅的数据看作prop。我们把这个函数命名为withSubscription:

```javascript
const CommentListWithSubscription = withSubscription(
  CommentList,
  (DataSource) => DataSource.getComments()
);

const BlogPostWithSubscription = withSubscription(
  BlogPost,
  (DataSource, props) => DataSource.getBlogPost(props.id)
});
```

第一个参数是需要被包裹的组件。第二个参数取回我们感兴趣的数据的函数，该函数的参数是DataSource和当前props。

当CommentListWithSubscription和BlogPostSubscription被渲染时，CommentList和BlogPost将被传递prop数据，该prop数据是从DataSource抽取出的最新数据：

```javascript
// This function takes a component...
function withSubscription(WrappedComponent, selectData) {
  // ...and returns another component...
  return class extends React.Component {
    constructor(props) {
      super(props);
      this.handleChange = this.handleChange.bind(this);
      this.state = {
        data: selectData(DataSource, props)
      };
    }

    componentDidMount() {
      // ... that takes care of the subscription...
      DataSource.addChangeListener(this.handleChange);
    }

    componentWillUnmount() {
      DataSource.removeChangeListener(this.handleChange);
    }

    handleChange() {
      this.setState({
        data: selectData(DataSource, this.props)
      });
    }

    render() {
      // ... and renders the wrapped component with the fresh data!
      // Notice that we pass through any additional props
      return <WrappedComponent data={this.state.data} {...this.props} />;
    }
  };
}
```
注意HOC不修改输入组件，也不使用继承来复制其行为。相反，HOC通过包裹原始组件到container组件中的方式来构建原始组件。HOC是无副作用的纯函数。

其实就是这样！被包裹的组件接收Container组件的所有props，同时被包裹的组件也有新的prop，data。被包裹组件用props来渲染输出。HOC并不关心数据为什么和怎么使用数据，被包裹的组件也不会关心数据从哪里来。

因为withSubscription是普通函数，所以你可以按你的需要添加参数。比如，你也许想要使prop data变得可配置，以此来使HOC和被包裹的组件解偶合。或者你可能会接收一个配置shouldComponentUpdate的参数，或者一个配置数据源的参数：这些都是可能的，因为HOC可以完全控制怎么定义组件。

如同组件，withSubscription和被包裹的组件之间完全基于prop连接。只要HOC对被包裹的组件提供一样的props，那么不同HOC的交换是容易的。举个例子，这对于你切换不同的数据请求库是有用的。

不要转变原有组件。使用构建。
反对在HOC内部临时改变某组件的prototype或其他改变。

```javascript
function logProps(InputComponent) {
  InputComponent.prototype.componentWillReceiveProps(nextProps) {
    console.log('Current props: ', this.props);
    console.log('Next props: ', nextProps);
  }
  // The fact that we're returning the original input is a hint that it has
  // been mutated.
  return InputComponent;
}

// EnhancedComponent will log whenever props are received
const EnhancedComponent = logProps(InputComponent);
```

这样做会导致一些问题。其中之一是组件不能脱离增强组件再被复用。更严重的情况，如果你应用另一个改变被包裹组件componentWillReceiveProps的HOC组件到EnhancedComponent，那么第一个HOC的功能将被覆盖！这个HOC将不会以函数组件的方式起作用，其没用生命周期函数。

改变HOC是有缺陷的抽象--使用者它们是如何被实现的来避免HOC间的冲突。
不是修改组件，HOC应该使用构建，通过包裹输入组件到container组件的方式：

```javascript
function logProps(WrappedComponent) {
  return class extends React.Component {
    componentWillReceiveProps(nextProps) {
      console.log('Current props: ', this.props);
      console.log('Next props: ', nextProps);
    }
    render() {
      // Wraps the input component in a container, without mutating it. Good!
      return <WrappedComponent {...this.props} />;
    }
  }
}
```
这个HOC和上面版本的HOC实现的功能一样，然而确避免了潜在的冲突。其和类和函数组件的工作方式一样好。由于它是纯函数，其可以和其他HOC甚至自身组合。

你可能注意到了HOC和container组件之间的相似性。container组件分开高层和底层关心功能的策略的一部分。container组件管理像订阅和state这类的事，并传递props给负责渲染UI的组件。HOC使用container组件作为它实现的一部分。你可以认为HOC是参数化的container组件定义。

## 公约：包含展示名字方便调试

HOC创建的container组件在React Developer Tool中的展现和所有组件一样。为了方便调试，选择某个展示名字以便表达其是哪个HOC产生的组件。

最常用的技巧是包含被包裹组件的名字。因此如果你的高阶组件被命名为withSubscription，并且被包裹的组件的名字叫做CommentList，可以使用展示名字withSubscription（CommentList）：
```javascript
render() {
  // Filter out extra props that are specific to this HOC and shouldn't be
  // passed through
  const { extraProp, ...passThroughProps } = this.props;

  // Inject props into the wrapped component. These are usually state values or
  // instance methods.
  const injectedProp = someStateOrInstanceMethod;

  // Pass props to wrapped component
  return (
    <WrappedComponent
      injectedProp={injectedProp}
      {...passThroughProps}
    />
  );
}
```

## 警告
对于react新手，高阶组件有一些并不显而易见的警告。

### 在render方法中不要使用HOC

React的diffing算法（如recomciliation）使用组件恒等来决定是否react应该更新存在的子树或者丢弃并重新挂载一个。如果从render函数中返回的组件和之前返回的组件是恒等的，React会递归地使用diffing来更新子树中的下一个组件。如果它们不等，之前的子树将会完全卸载。
通常，你不需要考虑这一点。但对HOC却需要在乎，因为在组件的render方法中不要使用HOC来生成组件：
```javascript
render() {
  // A new version of EnhancedComponent is created on every render
  // EnhancedComponent1 !== EnhancedComponent2
  const EnhancedComponent = enhance(MyComponent);
  // That causes the entire subtree to unmount/remount each time!
  return <EnhancedComponent />;
}
```
这里并不仅仅是性能问题--重新加载组件引发该组件和所有子组件state的丢失。
相反，在组件外使用HOC来保障结果组件仅被定义一次。那时，在各render方法中组件是一致恒等的。无论如何这都是我们想要的。

在很少的场景中你需要动态的使用HOC，你也可以在组件的生命周期函数或构造函数中使用它。

## 静态方法必须被向上拷贝

有时在React组件中定义静态方法是有用的。比如，container暴露一个静态方法getFragment来加快GraphQL碎片的组合。

当你对某组件使用HOC时，本质上是原始组件被包裹在container组件中。那意味着新的组件不会用原始组件的任何静态方法。

```javascript
// Define a static method
WrappedComponent.staticMethod = function() {/*...*/}
// Now apply an HOC
const EnhancedComponent = enhance(WrappedComponent);

// The enhanced component has no static method
typeof EnhancedComponent.staticMethod === 'undefined' // true
To solve this, you could copy the methods onto the container before returning it:

function enhance(WrappedComponent) {
  class Enhance extends React.Component {/*...*/}
  // Must know exactly which method(s) to copy :(
  Enhance.staticMethod = WrappedComponent.staticMethod;
  return Enhance;
}
```
为了解决这个问题，你可以复制这些方法到container组件上在返回组件前。

然而，这要求明确的知道那些方法需要拷贝。你可以使用hoist-non-react-statics来自动拷贝所有非React静态方法：

```javascript
import hoistNonReactStatic from 'hoist-non-react-statics';
function enhance(WrappedComponent) {
  class Enhance extends React.Component {/*...*/}
  hoistNonReactStatic(Enhance, WrappedComponent);
  return Enhance;
}
```

另外一种可行的解决方案是组件自身单独的导出静态方法。

```javascript
// Instead of...
MyComponent.someFunction = someFunction;
export default MyComponent;

// ...export the method separately...
export { someFunction };

// ...and in the consuming module, import both
import MyComponent, { someFunction } from './MyComponent.js';
```

### refs不能被传递下去
尽管高阶组件的公约是传递所有的props给被包裹的组件，但传递refs是不太可能的。那是因为ref不是正真的prop，如key，其被React特别处理。如果你添加ref到element，该element是HOC的结果，那么ref引用的是外层container的实例，不是被包裹组件的实例。

如果你发现你自己面临这样的问题，理想的方案是明确如何避免使用ref。偶尔，React新手在prop工作很好的情况下会依赖ref。

那就是说，有时需要refs是一个必要的逃跑窗口--然而react还不支持它们。input组件聚焦是你想要迫切控制的组件的一个例子。在那种情况下，一种解决方案是是把ref回调函数作为普通的prop（通过将其命名为不同的component）传递:

```javascript
function Field({ inputRef, ...rest }) {
  return <input ref={inputRef} {...rest} />;
}

// Wrap Field in a higher-order component
const EnhancedField = enhance(Field);

// Inside a class component's render method...
<EnhancedField
  inputRef={(inputEl) => {
    // This callback gets passed through as a regular prop
    this.inputEl = inputEl
  }}
/>

// Now you can call imperative methods
this.inputEl.focus();
```

这绝不是一个完美的解决方案。我们更倾向于库来关心ref而不是要求你手工来处理他们。我们正探索解决这个问题的方式以便对于使用HOC来说是透明的。
