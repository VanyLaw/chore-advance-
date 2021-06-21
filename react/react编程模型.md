## [Host tree](https://overreacted.io/zh-hans/react-as-a-ui-runtime/) ##
React 程序通常会输出一棵会随时间变化的树。它有可能是一棵 DOM 树 ，iOS 视图层 ，PDF 原语 ，又或是 JSON 对象 。然而，通常我们希望用它来展示 UI 。我们称它为“宿主树”，因为它往往是 React 之外宿主环境中的一部分 — 就像 DOM 或 iOS 。宿主树通常有它自己的命令式 API 。而 React 就是它上面的那一层。

所以到底 React 有什么用呢？非常抽象地，它可以帮助你编写可预测的，并且能够操控复杂的宿主树进而响应像用户交互、网络响应、定时器等外部事件的应用程序。

## host instance ##
宿主树(Host tree)由节点组成，我们称之为“宿主实例”(host instance)。在 DOM 环境中，宿主实例就是我们通常所说的 DOM 节点。宿主实例有它们自己的属性（例如 domNode.className 或者 view.tintColor ）。它们也有可能将其他的宿主实例作为子项。

通常会有原生的 API 用于操控这些宿主实例。例如，在 DOM 环境中会提供像 appendChild、removeChild、setAttribute 等一系列的 API 。在 React 应用中，通常你不会调用这些 API ，因为那是 React 的工作。

## renders ##
渲染器教会 React 如何与特定的宿主环境通信以及如何管理它的宿主实例。React DOM、React Native 甚至 Ink 都可以称作 React 渲染器。你也可以创建自己的 React 渲染器 。

react renderer主要工作在2种模式上：
- 可变模式（mutating）
大部分的renderer是可变模式，如DOM，host instance完全可变。

- 不可变模式（persistent）
适用于不提供appendChild等API，而是克隆parent tree并替换顶级子树的宿主环境。不可变性使得多线程变得更加容易，react fabric（fluent UI）就是利用这一模式。

## react element ##
在宿主环境中，一个宿主实例（例如 DOM 节点）是最小的构建单元。而在 React 中，最小的构建单元是 React 元素。React 元素是一个普通的 JavaScript 对象。它用来描述一个宿主实例。React 元素是轻量级的因为没有宿主实例与它绑定在一起。

```
// JSX 是用来描述这些对象的语法糖。
// <button className="blue" />
{
  type: 'button',
  props: { className: 'blue' }
}
```

就像宿主实例一样，React 元素也能形成一棵树。**React 元素具有不可变性**，你不能改变 React 元素中的子元素或者属性。如果你想要在稍后渲染一些不同的东西，你需要从头创建新的 React 元素树来描述它。我喜欢将 React 元素比作电影中放映的每一帧。它们捕捉 UI 在特定的时间点应该是什么样子。它们永远不会再改变。

## Entry Point ##
每一个 React 渲染器都有一个“入口”。正是那个特定的 API 让我们告诉 React ，将特定的 React 元素树渲染到真正的宿主实例中去。例如，React DOM 的入口就是 ReactDOM.render ：

```
ReactDOM.render(
  // { type: 'button', props: { className: 'blue' } }
  <button className="blue" />,
  document.getElementById('container')
);

// 在 ReactDOM 渲染器内部（简化版）
function createHostInstance(reactElement) {
  let domNode = document.createElement(reactElement.type);
  domNode.className = reactElement.props.className;
  return domNode;
}
const domNode = createHostInstance(<button ...>);
domContainer.appendChild(domNode);
```
如果 React 元素在 reactElement.props.children 中含有子元素，React 会在第一次渲染中递归地为它们创建宿主实例。

## Reconciliation ##
如果我们用同一个 container 调用 ReactDOM.render() 两次会发生什么呢？
```
ReactDOM.render(
  <button className="blue" />,
  document.getElementById('container')
);

// ... 之后 ...

// 应该替换掉 button 宿主实例吗？
// 还是在已有的 button Dom上更新属性？
ReactDOM.render(
  <button className="red" />,
  document.getElementById('container')
);
```
React 的工作是根据 React 元素树构建宿主树（即create DOM）。对宿主实例（DOM）做什么操作来响应新的信息叫做协调（reconciliation）。有2种方法解决上面的情况，简化版的React会丢弃已存在的树，然后重新开始创建。
```
let domContainer = document.getElementById('container');
// 清除掉原来的树
domContainer.innerHTML = '';
// 创建新的宿主实例树
let domNode = document.createElement('button');
domNode.className = 'red';
domContainer.appendChild(domNode);
```

但是在 DOM 环境下，这样的做法效率低下而且会丢失像 focus、selection、scroll 等许多状态。相反，我们希望 React 这样做：
```
let domNode = domContainer.firstChild;
// 更新已有的宿主实例
domNode.className = 'red';
```
所以， **如果相同的元素类型在同一个地方先后出现两次，React 会重用已有的宿主实例。**

```
// let domNode = document.createElement('button');
// domNode.className = 'blue';
// domContainer.appendChild(domNode);
ReactDOM.render(
  <button className="blue" />,
  document.getElementById('container')
);

// 能重用宿主实例吗？能！(button → button)
// domNode.className = 'red';
ReactDOM.render(
  <button className="red" />,
  document.getElementById('container')
);

// 能重用宿主实例吗？不能！(button → p)
// domContainer.removeChild(domNode);
// domNode = document.createElement('p');
// domNode.textContent = 'Hello';
// domContainer.appendChild(domNode);
ReactDOM.render(
  <p>Hello</p>,
  document.getElementById('container')
);

// 能重用宿主实例吗？能！(p → p)
// domNode.textContent = 'Goodbye';
ReactDOM.render(
  <p>Goodbye</p>,
  document.getElementById('container')
);
```
所以，react element会重新构造，但DOM元素可能会重用。

## 条件语句 ##
如果 React 在渲染更新前后只重用那些元素类型匹配的宿主实例，那当遇到包含条件语句的内容时又该如何渲染呢？
```
// 第一次渲染
ReactDOM.render(
  <dialog>
    <input />
  </dialog>,
  domContainer
);

// 下一次渲染
ReactDOM.render(
  <dialog>
    <p>I was just added here!</p>
    <input />
  </dialog>,
  domContainer
);

// input -> p  类型改变，remove input，然后创建p
let oldInputNode = dialogNode.firstChild;
dialogNode.removeChild(oldInputNode);

let pNode = document.createElement('p');
pNode.textContent = 'I was just added here!';
dialogNode.appendChild(pNode);

// nothing -> input, 创建新的input
let newInputNode = document.createElement('input');
dialogNode.appendChild(newInputNode);
```
这样的做法并不科学因为事实上 ``<input>`` 并没有被 ``<p>`` 所替代 — 它只是移动了位置而已。我们不希望因为重建 DOM 而丢失了 selection、focus 等状态以及其中的内容。

但下面的例子却不会遇到这个问题：

```
function Form({ showMessage }) {
  let message = null;
  if (showMessage) {
    message = <p>I was just added here!</p>;
  }
  // 不管showMessage是true or false，在渲染过程中<input />总是第二个孩子。
  return (
    <dialog>
      {message}
      <input />
    </dialog>
  );
}

// dialog -> dialog 重用
//    null/p -> p 创建或重用
//    input -> input 重用
let inputNode = dialogNode.firstChild;
let pNode = document.createElement('p');
pNode.textContent = 'I was just added here!';
dialogNode.insertBefore(pNode, inputNode);
```

## Lists ##
比较树中同一位置的元素类型对于是否该重用还是重建相应的宿主实例往往已经足够。但这只适用于当子元素是静止的并且不会重排序的情况。而当遇到动态列表时，我们不能确定其中的顺序总是一成不变的。

如果只是简单的使用index作为key，当重排时，React 只会对其中的每个元素进行更新而不是将其重新排序。这样做会造成性能上的问题和潜在的 bug 。例如，当商品列表的顺序改变时，原本在第一个输入框的内容仍然会存在于现在的第一个输入框中。因为在每个react instance（DOM）里的其他属性没变。

需要注意的是 key 只与特定的父亲 React 元素相关联。React 并不会去匹配父元素不同但 key 相同的子元素。（React 并没有惯用的支持对在不重新创建元素的情况下让宿主实例在不同的父元素之间移动。）

## Purity ##
React 组件中对于 props 应该是纯净的。
```
function Button(props) {
  // 🔴 没有作用
  props.isActive = true;
}
```

但局部改变时允许的：当我们在函数组件内部创建 items 时不管怎样改变它都行，只要这些改变发生在将其作为最后的渲染结果之前。所以并不需要重写你的代码来避免局部mutation。

```
function FriendList({ friends }) {
  let items = [];
  for (let i = 0; i < friends.length; i++) {
    let friend = friends[i];
    items.push(
      <Friend key={friend.id} friend={friend} />
    );
  }
  return <section>{items}</section>;
}
```

同样地，惰性初始化是被允许的即使它不是完全“纯净”的：
```
function ExpenseForm() {
  // 只要不影响其他组件这是被允许的：
  SuperCalculator.initializeIfNotReady();

  // 继续渲染......
}
```

只要调用组件多次是安全的，并且不会影响其他组件的渲染，React 并不关心你的代码是否像严格的函数式编程一样百分百纯净。在 React 中，幂等性比纯净性更加重要。

## Recursion递归 ##
从Element构建DOM的时候会递归执行（以前）。
```
你： ReactDOM.render(<App />, domContainer)
React： App ，你想要渲染什么？

App ：我要渲染包含 <Content> 的 <Layout> 。
React： <Layout> ，你要渲染什么？

Layout ：我要在 <div> 中渲染我的子元素。我的子元素是 <Content> 所以我猜它应该渲染到 <div> 中去。
React： <Content> ，你要渲染什么？

<Content> ：我要在 <article> 中渲染一些文本和 <Footer> 。
React： <Footer> ，你要渲染什么？

<Footer> ：我要渲染含有文本的 <footer> 。
React： 好的，让我们开始吧：

// 最终的 DOM 结构
<div>
  <article>
    Some text
    <footer>some more text</footer>
  </article>
</div>
```
这就是为什么我们说协调是递归式的。当 React 遍历整个元素树时，可能会遇到元素的 type 是一个组件。React 会调用它然后继续沿着返回的 React 元素下行。最终我们会调用完所有的组件，然后 React 就会知道该如何改变宿主树。

## 控制反转 ##
你也许会好奇：为什么我们不直接调用组件？为什么要编写 ``<Form />`` 而不是 ``Form()`` ？

React 能够做的更好如果它“知晓”你的组件而不是在你递归调用它们之后生成的 React 元素树。
```
// 🔴 React 并不知道 Layout 和 Article 的存在。
// 因为你在调用它们。
ReactDOM.render(
  Layout({ children: Article() }),
  domContainer
)

// ✅ React知道 Layout 和 Article 的存在。
// React 来调用它们。（控制反转）
ReactDOM.render(
  <Layout><Article /></Layout>,
  domContainer
)
```
- 组件不仅仅只是函数。 React 能够用local state等特性来增强组件功能。如果你直接调用了组件，你就只能自己来维护state的更新了。

- 组件类型参与协调。React会帮助进行实例（DOM）的复用。例如，当你从渲染 <Feed> 页面转到 Profile 页面，React 不会尝试重用其中的宿主实例 — 就像你用 <p> 替换掉 <button> 一样。所有的状态都会丢失 — 对于渲染完全不同的视图时，通常来说这是一件好事。你不会想要在 <PasswordForm> 和 <MessengerChat> 之间保留输入框的状态尽管 <input> 的位置意外地“排列”在它们之间。

- React 能够推迟协调。 如果让 React 控制调用你的组件，它能做很多有趣的事情。例如，它可以让浏览器在组件调用之间做一些工作，这样重渲染大体量的组件树时就不会阻塞主线程。想要手动编排这个过程而不依赖 React 的话将会十分困难。

- 更好的可调试性。

## Lazy Evaluation惰性求值 ##
让 React 来决定何时以及是否调用组件。如果我们的的 Page 组件忽略自身的 children prop 且相反地渲染了 <h1>Please login</h1> ，React 不会尝试去调用 Comments 函数。
```
function Page({ currentUser, children }) {
  if (!currentUser.isLoggedIn) {
    return <h1>Please login</h1>;
  }
  return (
    <Layout>
      {children}
    </Layout>
  );
}

function Story({ currentUser }) {
  return (
    <Page user={currentUser}>
      <Comments />
    </Page>
  );
}
```
这很好，因为它既可以让我们避免不必要的渲染也能使我们的代码变得不那么脆弱。（当用户退出登录时，我们并不在乎 Comments 是否被丢弃 — 因为它从没有被调用过。）能减少无用组件的创建和销毁。

## Consistency一致性 ##
即使我们想将协调过程本身分割成非阻塞的工作块，我们仍然需要在同步的循环中对真实的宿主实例进行操作。这样我们才能保证用户不会看见半更新状态的 UI ，浏览器也不会对用户不应看到的中间状态进行不必要的布局和样式的重新计算。

这也是为什么 React 将所有的工作分成了“渲染阶段”和“提交阶段”的原因。渲染阶段 是当 React 调用你的组件然后进行协调的时段（异步的）。**提交阶段 就是 React 操作宿主树的时候。而这个阶段永远是同步的**。

## 缓存 ##
当父组件通过 setState 准备更新时，**React 默认会协调整个子树**。因为 React 并不知道在父组件中的更新是否会影响到其子代，所以 React 默认保持一致性。这听起来会有很大的性能消耗但事实上对于小型和中型的子树来说，这并不是问题。

当树的深度和广度达到一定程度时，你可以让 React 去缓存子树并且重用先前的渲染结果，当 prop 在浅比较之后是相同时。默认情况下，React 不会故意缓存组件。
```
function Row({ item }) {
  // ...
}

export default React.memo(Row); // 组件的缓存
```
现在，在父组件 <Table> 中调用 setState 时如果 <Row> 中的 item 与先前渲染的结果是相同的，React 就会直接跳过协调的过程。

你可以通过 useMemo() Hook 获得**单个表达式级别的细粒度缓存。该缓存于其相关的组件紧密联系在一起，并且将与局部状态一起被销毁**。它只会保留最后一次计算的结果。

## 原始模型Raw Models ##
React 并没有使用“反应式”的系统来支持细粒度的更新。换句话说，任何在顶层的更新只会触发协调（整体更新）而不是局部更新那些受影响的组件。（重新创建元素树）

这样的设计是有意而为之的。对于 web 应用来说交互时间是一个关键指标，而通过遍历整个模型去设置细粒度的监听器只会浪费宝贵的时间。此外，在很多应用中交互往往会导致或小（按钮悬停）或大（页面转换）的更新，因此细粒度的订阅只会浪费内存资源。

React 的设计原则之一就是它可以处理原始数据。如果你拥有从网络请求中获得的一组 JavaScript 对象，你可以将其直接交给组件而无需进行预处理。没有关于可以访问哪些属性的问题，或者当结构有所变化时造成的意外的性能缺损。React 渲染是 O(视图大小) 而不是 O(模型大小) ，并且你可以通过 windowing 显著地减少视图大小。

‘windowing’的技术把大数据集分解为多个块，当它们滚动到视图中时可以即时加载。它也可以用于创建无限加载列表。例如：react-virtualized，react-window等都使用了“windowing”的技术。

## 批量更新Batch update ##
一些组件也许想要更新状态来响应同一事件。
```
function Parent() {
  let [count, setCount] = useState(0);
  return (
    <div onClick={() => setCount(count + 1)}>
      Parent clicked {count} times
      <Child />
    </div>
  );
}

function Child() {
  let [count, setCount] = useState(0);
  return (
    <button onClick={() => setCount(count + 1)}>
      Child clicked {count} times
    </button>
  );
}

// 点击button后，子组件的 onClick 首先被触发（同时触发了它的 setState ）。（冒泡）然后父组件在它自己的 onClick 中调用 setState 。
```
如果 React 立即重渲染组件以响应 setState 调用，最终我们会重渲染子组件两次：
```
*** 进入 React 浏览器 click 事件处理过程 ***
Child (onClick)
  - setState
  - re-render Child // 😞 不必要的重渲染
Parent (onClick)
  - setState
  - re-render Parent
  - re-render Child
*** 结束 React 浏览器 click 事件处理过程 ***
```
第一次 Child 组件渲染是浪费的。并且我们也不会让 React 跳过 Child 的第二次渲染因为 Parent 可能会传递不同的数据由于其自身的状态更新。

**这就是为什么 React 会在组件内所有事件触发完成后再进行批量更新的原因：**
```
*** 进入 React 浏览器 click 事件处理过程 ***
Child (onClick)
  - setState
Parent (onClick)
  - setState
*** Processing state updates                     ***
  - re-render Parent
  - re-render Child
*** 结束 React 浏览器 click 事件处理过程  ***
```
*组件内调用 setState 并不会立即执行重渲染。相反，React 会先触发所有的事件处理器，然后再触发一次重渲染以进行所谓的批量更新。*

当状态逻辑变得更加复杂而不仅仅只是少数的 setState 调用时，我建议你使用 useReducer Hook 来描述你的局部状态。它就像 “updater” ( setState(state=> state+1) )的升级模式在这里你可以给每一次更新命名：

```
const [counter, dispatch] = useReducer((state, action) => {
    if (action === 'increment') {
      return state + 1;
    } else {
      return state;
    }
  }, 0);

  function handleClick() {
    dispatch('increment'); // 如果都是setState(counter+1); 因为异步更新，最终只会+1
    dispatch('increment');
    dispatch('increment');
  }
```

## 调用树Call tree ##
React 内部有自己的调用栈用来记忆我们当前正在渲染的组件，例如 [App, Page, Layout, Article /* 此刻的位置 */]。React 与通常意义上的编程语言进行时不同，因为它目标是渲染 UI 树。这些树需要保持“活性”，这样才能使我们与其进行交互。在第一次 ReactDOM.render() 执行后，DOM不会消失。

React 组件可以看成是call tree（调用树）而不是call stack（调用栈）。当我们调用完 Article 组件，它的 React “调用树” 帧并没有被摧毁。我们需要将local state保存以便映射到宿主实例的某个地方。

这些“调用树”帧会随它们的局部状态和宿主实例一起被摧毁，但是只会在协调规则认为这是必要的时候执行。如果你曾经读过 React 源码，你就会知道这些帧其实就是 Fibers 。

Fibers 是局部状态真正存在的地方。当状态被更新后，React 将其下面的 Fibers 标记为需要进行协调，之后便会调用这些组件。

## Hooks 调用顺序 ##
Hooks 的内部实现其实是链表 。当你调用 useState 的时候，我们将指针移到下一项。当我们退出组件的“调用树”帧时，会缓存该结果的列表直到下次渲染开始。
### Only Call Hooks at the Top Level ###
- Don’t call Hooks inside loops, conditions, or nested functions. 

### Only Call Hooks from React Functions ###
- Don’t call Hooks from regular JavaScript functions.

```
let hooks, i;
function useState(){
  i++;
  if(hooks[i]){
    // 再次渲染时
    return hooks[i];
  }
  // 第一次渲染
  hooks.push(...)
}

// 准备渲染
i=-1;
hooks = fiber.hooks || [];
// 调用组件
YourComponent();
// 缓存 Hooks 状态
fiber.hooks = hooks;

```