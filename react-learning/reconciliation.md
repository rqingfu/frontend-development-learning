# Reconciliation

React提供声明式api以便你不用担心在每次更新时确切地改变了哪些部分。这是编写应用更加容易，但是这使得在React内是如何实现变得不显而易见。这篇文章描述如何选择React"diffing"算法以便组件更新是可预测的，同时对于高性能应用足够快。

## 动机

当你使用React时，你可以认为render函数在某个时间点上创建了一棵React element树。下一次state或prop更新时，render函数会返回一棵不同的React element树。React之后需要识别如何有效地更新UI，该UI和最新的React element树是匹配的。

对于产生从一棵树转换为另一棵树的最少操作步骤数问题是有一般的解决方案的。然而，经典算法的复杂的是O(n3), n是该树上的element数量。

如果我们在React中使用该算法，展示1000 element需要做十亿次比较。这代价太昂贵了。相反，React基于如下两个假设React实现了一个O(n)的启发式算法：

1、 不同类型的两元素将产生不同的树。
2、 开发者可以通过是用key prop暗示在每次渲染中哪个子元素是稳定的。

在现实中，上面的假设几乎对所有真实使用场景是有效的。

## Diffing算法

当转换两个树时，React首先比较两个根元素。依据根元素类型会用不同的行为。

### 不同类型的元素

当根元素有不同的类型时，React将拆卸旧的element树并且重新构建新的element树。如从&lt;a>到&lt;img>, 或者从&lt;Article>到&lt;Comment>, 或者从&lt;Button>到&lt;div> - 这些都会导致重新完全构建树。

当卸载树时，旧DOM节点将被摧毁。组件实例执行componentWillUnmount()函数。 但构建一棵新的树时，新的DOM节点被插入DOM树种。组件实例将依次执行componentWillMount()和componentDidMount()函数。任何和旧树相关的state丢失。

根节点下面的任何组件也将会被卸载并让它们的state也被摧毁。比如，当转换如下元素时：
```
<div>
  <Counter />
</div>

<span>
  <Counter />
</span>
```
这会卸载旧的Counter并重新挂载新的Counter。

### 同类型的DOM元素

当比较两个类型相同的React DOM元素时，React优先保留相同的DOM节点，然后查看两个元素的属性(attributes)并仅更新改变的属性。例如：
```
<div className="before" title="stuff" />

<div className="after" title="stuff" />
```

通过比较这两个元素，React知道只需在原来DOM节点上修改className。

当更新style属性时， React也知道仅仅更新改变的properties。例如:
```
<div style={{color: 'red', fontWeight: 'bold'}} />

<div style={{color: 'green', fontWeight: 'bold'}} />
```
当转换这两个元素时，React知道只需修改property color样式，不需要改变fontWeight。

处理完DOM节点后，React接着递归转换该元素的子节点。

### 同类型的组件

当组件更新时，组件实例保持不变以便state在render时保持不变。React更新保留组件实例的props来匹配新的element，并且调用该组件实例的componentWillReceiveProps()和componentWillUpdate()。

接下来，render()方法被调用并且diff算法在之前结果和新结果上做递归。

### 在孩子节点上递归

通常，当在DOM节点上递归时，React在同一时间仅仅在孩子节点列表上迭代并且但有不同是产生变化。

例如，当在孩子节点的末尾添加元素时，两棵树的转换会工作的很好：
```
<ul>
  <li>first</li>
  <li>second</li>
</ul>

<ul>
  <li>first</li>
  <li>second</li>
  <li>third</li>
</ul>
```
React将匹配两个&lt;li>first&lt;/li>树, 匹配两个&lt;li>second&lt;/li>树, 接着插入&lt;li>third&lt;/li>树。

如果你随意的实现，在孩子节点的开始出添加元素会导致很差的性能。比如，转换这两棵树的性能很差：
```
<ul>
  <li>Duke</li>
  <li>Villanova</li>
</ul>

<ul>
  <li>Connecticut</li>
  <li>Duke</li>
  <li>Villanova</li>
</ul>
```
React将会更改每个子节点而不是完整地保留&lt;li>Duke&lt;/li>和&lt;li>Villanova&lt;/li>s树。无效率是有问题的。

### Keys

为了解决上述问题，React支持key属下。当孩子节点用了key，React使用key来匹配原来树的子节点和接下来树的子节点。例如：给上面低效的例子添加key来高效的转换子节点：
```
<ul>
  <li key="2015">Duke</li>
  <li key="2016">Villanova</li>
</ul>

<ul>
  <li key="2014">Connecticut</li>
  <li key="2015">Duke</li>
  <li key="2016">Villanova</li>
</ul>
```
现在React知道key为2014的元素是一个新元素，key 2015和key 2016的元素仅仅被移动。

在实际应用中，查找key通常不是很难。你将展现的元素可能已经用了唯一ID，该key来自你的数据：
```
<li key={item.id}>{item.name}</li>
```
当数据中没有id时，你可以在你的模型中添加新的ID或者哈希部分内容来产生key。key只需要在兄弟节点中是唯一的，不需要全局唯一。

万不得已，你可以传递数组中数据的索引作为key。如果数组中的数据从不重组，这种方式的效率不错，但是重组将会变慢。

### 折衷

重要的是要记住,reconciliation算法是一个实现细节。React可能重绘整个app；最后的结果可能是一样的。我们将定期改善启发法以便使常见案例更快。

在目前的实现中，你可以

认为子树在其兄弟节点中已经被移动，但是你不能区分它已经移动什么地方。算法将重新渲染整个子树。


因为React依赖启发法，如果这些假设没有被使用，性能将会受到影响。

1、算法不会尝试匹配不同组件类型的中的子树。如果你发现需要替换组件类型其组件的子组件一样，你可能需要使其具体相同的类型。在实际应用中，我们没有发现这是一个问题。

2、Key应该是稳定的，可预测的和唯一的。不稳定的key(像通过Math.random()产生的key)将导致许多组件实例和DOM节点，这些节点很多都不会被再创建，这会引起性能下降和子组件state的丢失。