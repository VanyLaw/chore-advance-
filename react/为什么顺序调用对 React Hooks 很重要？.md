考虑以下的替换方案

# 缺陷1：无法提取 custom hook #
例如：有个替代方案是限制一个组件调用多次 useState()，你可以把 state 放在一个对象里，这样还可以兼容 class 不是更好吗？
```
function Form() {
  const [state, setState] = useState({
    name: 'Mary',
    surname: 'Poppins',
    width: window.innerWidth,
  });
  // ...
}
```
要清楚，Hooks 是允许这种风格写的，你不必将 state 拆分成一堆 state 变量.

但支持多次调用 useState() 的关键在于，你可以从组件中提取出部分有状态逻辑（state + effect）到 custom hooks 中，同时可以单独使用本地 state 和 effects：
```
function Form() {
  // 在组件内直接定义一些 state 变量
  const [name, setName] = useState('Mary');
  const [surname, setSurname] = useState('Poppins');

  // 我们将部分 state 和 effects 移至 custom hook
  const width = useWindowWidth();
  // ...
}

function useWindowWidth() {
  // 在 custom hook 内定义一些 state 变量
  const [width, setWidth] = useState(window.innerWidth);
  useEffect(() => {
    // ...
  });
  return width;
}
```
如果你只允许每个组件调用一次 useState()，你将失去用 custom hook 引入 state 能力，这就是 custom hooks 的关键。

# 缺陷 2: 命名冲突 #
一个常见的建议是让组件内 useState() 接收一个唯一标识 key 参数（string 等）区分 state 变量。这试图摆脱依赖顺序调用，但引入了另外一个问题 —— 命名冲突。
```
// ⚠️ 这不是 React Hooks API
function Form() {
  // 我们传几种 state key 给 useState()
  const [name, setName] = useState('name');
  const [surname, setSurname] = useState('surname');
  const [width, setWidth] = useState('width');
  // ...
```
当然除了错误之外，你可能无法在同一个组件调用两次 useState('name')，这种偶然发生的可以归结于其他任意 bug，但是，当你使用一个 custom hook 时，你总会遇到想添加或移除 state 变量和 effects 的情况。

这个提议中，每当你在 custom hook 里添加一个新的 state 变量时，就有可能破坏使用它的任何组件（直接或者间接），因为 可能已经有同名的变量 位于组件内。**当前代码可能看起来总是「优雅的」，但应对需求变化时十分脆弱**

实际中 Hooks 提案通过依赖顺序调用来解决这个问题：即使两个 Hooks 都用 name state 变量，它们也会彼此隔离，每次调用 useState() 都会获得独立的 「内存单元」。

# 缺陷 3：同一个 Hook 无法调用两次 #
给 useState 「加key」的另一种衍生提案是使用像 Symbol 这样的东西，这样就不冲突了对吧？
```
// ⚠️ 这不是 React Hooks API
const nameKey = Symbol();
const surnameKey = Symbol();
const widthKey = Symbol();

function Form() {
  // 我们传几种state key给useState()
  const [name, setName] = useState(nameKey);
  const [surname, setSurname] = useState(surnameKey);
  const [width, setWidth] = useState(widthKey);
  // ...

```
这个提案看上去对提取出来的 useWindowWidth Hook（customer hook）有效
```
// ⚠️ 这不是 React Hooks API
function Form() {
  // ...
  const width = useWindowWidth();
  // ...
}

/*********************
 * useWindowWidth.js *
 ********************/
const widthKey = Symbol();
 
function useWindowWidth() {
  const [width, setWidth] = useState(widthKey);
  // ...
  return width;
}
```
但如果尝试提取出来的 input handling，它会失败：
```
// ⚠️ 这不是 React Hooks API
function Form() {
  // ...
  const name = useFormInput();
  const surname = useFormInput();
  // ...
  return (
    <>
      <input {...name} />
      <input {...surname} />
      {/* ... */}
    </>    
  )
}

/*******************
 * useFormInput.js *
 ******************/
const valueKey = Symbol(); // 同一个key
 
function useFormInput() {
  const [value, setValue] = useState(valueKey);
  return {
    value,
    onChange(e) {
      setValue(e.target.value);
    },
  };
}
```
我们调用 useFormInput() 两次，但 useFormInput() 总是用同一个 key 调用 useState().

# 缺陷 4：钻石问题(多层继承问题) #
比如useWindowWidth() 和 useNetworkStatus() 这两个 custom hooks 可能要用像 useSubscription() 这样的 custom hook，如下：
```
function StatusMessage() {
  const width = useWindowWidth();
  const isOnline = useNetworkStatus();
  return (
    <>
      <p>Window width is {width}</p>
      <p>You are {isOnline ? 'online' : 'offline'}</p>
    </>
  );
}

function useSubscription(subscribe, unsubscribe, getValue) {
  const [state, setState] = useState(getValue());
  useEffect(() => {
    const handleChange = () => setState(getValue());
    subscribe(handleChange);
    return () => unsubscribe(handleChange);
  });
  return state;
}

function useWindowWidth() {
  const width = useSubscription(
    handler => window.addEventListener('resize', handler),
    handler => window.removeEventListener('resize', handler),
    () => window.innerWidth
  );
  return width;
}

function useNetworkStatus() {
  const isOnline = useSubscription(
    handler => {
      window.addEventListener('online', handler);
      window.addEventListener('offline', handler);
    },
    handler => {
      window.removeEventListener('online', handler);
      window.removeEventListener('offline', handler);
    },
    () => navigator.onLine
  );
  return isOnline;
}
```
custom hook 作者准备或停止使用另一个 custom hook 应该是要安全的，而不必担心它是否已在链中某处「被用过了」。
(作为反例，遗留的 React createClass() 的 mixins 不允许你这样做，有时你会有两个 mixins，它们都是你想要的，但由于扩展了同一个 「base」 mixin，因此互不兼容。)
```
       / useWindowWidth()   \                   / useState()  🔴 Clash
Status                        useSubscription() 
       \ useNetworkStatus() /                   \ useEffect() 🔴 Clash
```

# 缺陷 5：复制粘贴的主意被打乱 #



原文： https://overreacted.io/zh-hans/why-do-hooks-rely-on-call-order/