
## React生命周期

### 组件的生命周期
每个组件都包含 “生命周期方法”，你可以重写这些方法，以便于在运行过程中特定的阶段执行这些方法

![](https://gitarticle.oss-cn-shanghai.aliyuncs.com/javascript/images/react-lifecycles.png)

#### 挂载
当组件实例被创建并插入DOM中时，其生命周期调用顺序如下：

- `constructor()`   初始化 this.state
- `static getDerivedStateFromProps()` 
- `render()` 更新DOM来匹配渲染的输出
- `componentDidMount()`  输出被插入到DOM中后，React就会调用此生命周期方法

#### 更新
当组件的`props`或`state`发生变化时会触发更新。组件更新的生命周期调用顺序如下：

- static getDerivedStateFromProps()
- `shouldComponentUpdate()`
- `render()`
- getSnapshotBeforeUpdate()
- `componentDidUpdate()`

#### 卸载
当组件从 DOM 中移除时会调用如下方法：

- `componentWillUnmount()`

#### 错误处理
当渲染过程，生命周期，或子组件的构造函数中抛出错误时，会调用如下方法：

- `static getDerivedStateFromError()`
- `componentDidCatch()`

```javascript
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error) {
    // 更新 state 使下一次渲染可以显示降级 UI
    return { hasError: true };
  }

  componentDidCatch(error, info) {
    // "组件堆栈" 例子:
    //   in ComponentThatThrows (created by App)
    //   in ErrorBoundary (created by App)
    //   in div (created by App)
    //   in App
    logComponentStackToMyService(info.componentStack);
  }

  render() {
    if (this.state.hasError) {
      // 你可以渲染任何自定义的降级 UI
      return <h1>Something went wrong.</h1>;
    }

    return this.props.children;
  }
}
```
### 常用的生命周期方法

#### `render()`
`render()`方法是 class 组件中唯一必须实现的方法。

当`render`被调用时，它会检查`this.props`和`this.state`的变化并返回以下类型之一：

- React 元素。通常通过 JSX 创建
- 数组或 fragments。 使得 render 方法可以返回多个元素
- Portals。可以渲染子节点到不同的 DOM 子树中。
- 字符串或数值类型。它们在 DOM 中会被渲染为文本节点
- 布尔类型或 null。什么都不渲染。（主要用于支持返回 test && <Child /> 的模式，其中 test 为布尔类型。)

`render()` 函数应该为`纯函数`，这意味着在不修改组件 state 的情况下，每次调用时都返回相同的结果，并且它不会直接与浏览器交互。

如需与`浏览器`进行`交互`，请在 `componentDidMount()` 或其他生命周期方法中执行你的操作。保持 render() 为纯函数，可以使组件更容易思考。

如果 `shouldComponentUpdate()` 返回 `false`，则不会调用 `render()`

#### constructor()

如果不初始化 state 或不进行方法绑定，则不需要为 React 组件实现构造函数。

通常，在 React 中，构造函数仅用于以下两种情况：

- 通过给`this.state` 赋值对象来初始化内部 state。
- 为事件处理函数绑定实例

在 `constructor()` 函数中不要调用 setState() 方法。如果你的组件需要使用内部 state，请直接在构造函数中为 this.state 赋值初始 state：
```javascript
constructor(props) {
  super(props);
  // 不要在这里调用 this.setState()
  this.state = { counter: 0 };
  this.handleClick = this.handleClick.bind(this);
}
```

要避免在构造函数中引入任何副作用或订阅。如遇到此场景，请将对应的操作放置在 `componentDidMount`中。

避免将`props`的值复制给`state`！这是一个常见的错误：
```javascript
constructor(props) {
 super(props);
 // 不要这样做
 this.state = { color: props.color };
}
```
如此做毫无必要（你可以直接使用 `this.props.color`），同时还产生了 bug（更新 prop 中的 color 时，并不会影响 state）。

只有在你刻意`忽略 prop 更新`的情况下使用。此时，应将 prop 重命名为 `initialColor` 或 `defaultColor`。

**建议：完全可控的组件**
阻止上述问题发生的一个方法是，从组件里删除 state。如果 prop 里包含了 email，我们就没必要担心它和 state 冲突。我们甚至可以把 EmailInput 转换成一个轻量的函数组件：
```javascript
function EmailInput(props) {
  return <input onChange={props.onChange} value={props.email} />;
}
```

**建议：有 key 的非可控组件**
另外一个选择是让组件自己存储临时的state。在这种情况下，组件仍然可以从prop接收初始值，但是更改之后的值就和 prop 没关系了：

```javascript
class EmailInput extends Component {
  state = { email: this.props.defaultEmail };

  handleChange = event => {
    this.setState({ email: event.target.value });
  };

  render() {
    return <input onChange={this.handleChange} value={this.state.email} />;
  }
}
```
必要时，你可以修改它的 `key`，以强制`重置`其内部 state。
当 key 变化时， React 会创建一个新的而不是更新一个既有的组件

在这密码管理器的例子中，为了在不同的页面切换不同的值，我们可以使用 key 这个特殊的 React 属性。当 key 变化时， React 会创建一个新的而不是更新一个既有的组件。在这个示例里，当用户输入时，我们使用 user ID 当作 key 重新创建一个新的 email input 组件：
```javascript
<EmailInput
  defaultEmail={this.props.user.email}
  key={this.props.user.id}
/>
```

#### componentDidMount()

`componentDidMount()` 会在组件`挂载`后（插入DOM树中）立即调用。依赖于`DOM节点的初始化`应该放在这里。如需通过`网络请求`获取数据，此处是实例化请求的好地方。

这个方法是比较适合`添加订阅`的地方。如果添加了订阅，请不要忘记在 `componentWillUnmount()` 里取消订阅

你可以在 `componentDidMount()` 里直接调用 `setState()`。它将触发`额外渲染`，但此渲染会发生在浏览器更新屏幕之前。如此保证了即使在 `render()` 两次调用的情况下，用户也不会看到中间状态。请谨慎使用该模式，因为它会导致性能问题。通常，你应该在 constructor() 中初始化 state。如果你的渲染依赖于 DOM 节点的大小或位置，比如实现 modals 和 tooltips 等情况下，你可以使用此方式处理

### componentDidUpdate(prevProps, prevState, snapshot)
`componentDidUpdate()` 会在更新后会被立即调用。首次渲染不会执行此方法。

当组件更新后，可以在此处对 DOM 进行操作。如果你对更新前后的 props 进行了比较，也可以选择在此处进行网络请求。（例如，当 props 未发生变化时，则不会执行网络请求）。
```javascript
componentDidUpdate(prevProps) {
  // 典型用法（不要忘记比较 props）：
  if (this.props.userID !== prevProps.userID) {
    this.fetchData(this.props.userID);
  }
}
```
你也可以在 `componentDidUpdate()` 中直接调用 `setState()`，但请注意它**必须被包裹在一个条件语句**里，正如上述的例子那样进行处理，否则会导致**死循环**。它还会导致额外的重新渲染，虽然用户不可见，但会影响组件性能。

如果组件实现了 `getSnapshotBeforeUpdate()` 生命周期（不常用），则它的返回值将作为 `componentDidUpdate()` 的第三个参数 `snapshot` 参数传递。否则此参数将为 `undefined`。

如果 `shouldComponentUpdate()` 返回值为 `false`，则不会调用 `componentDidUpdate()`

#### componentWillUnmount()
`componentWillUnmount()` 会在组件卸载及销毁之前直接调用。在此方法中执行必要的**清理**操作，例如，清除 timer，取消网络请求或清除在 componentDidMount() 中创建的订阅等。

componentWillUnmount() 中不应调用 setState()，因为该组件将永远不会重新渲染。组件实例卸载后，将永远不会再挂载它。

#### shouldComponentUpdate(nextProps, nextState)
根据 `shouldComponentUpdate()` 的返回值，判断 React 组件的输出是否受当前 `state` 或 `props` 更改的影响。默认行为是 state 每次发生变化组件都会重新渲染。大部分情况下，你应该遵循默认行为。

当 `props` 或 `state` 发生变化时，`shouldComponentUpdate()` 会在渲染执行之前被调用。返回值默认为 true。首次渲染或使用 forceUpdate() 时不会调用该方法。

此方法仅作为性能优化的方式而存在。不要企图依靠此方法来`阻止`渲染，因为这可能会产生 bug。你应该考虑使用内置的 `PureComponent` 组件，而不是手动编写


### setState(updater, [callback])

`setState()` 将对组件 `state` 的更改排入队列，并通知 React 需要使用更新后的 state 重新渲染此组件及其子组件。这是用于更新用户界面以响应事件处理器和处理服务器数据的主要方式

将 `setState()` 视为`请求`而**不是立即更新组件**的命令。为了更好的感知性能，React 会**延迟**调用它，然后通过一次传递更新多个组件。React 并不会保证 state 的变更会立即生效。

`setState()`在合成事件和钩子函数中是异步的，在原生事件和`setTimeout`中是同步的。
`setState()` 并不总是立即更新组件。它会批量推迟更新。这使得在调用 setState() 后立即读取 this.state 成为了隐患。为了消除隐患，请使用 `componentDidUpdate` 或者 `setState` 的**回调函数**，这两种方式都可以保证在应用更新后触发。如需基于之前的 state 来设置当前的 state，请阅读下述关于参数 updater 的内容。

`setState()`在合成事件和钩子函数中是异步的，在原生事件和`setTimeout`中是同步的。

除非 `shouldComponentUpdate()` 返回 false，否则 setState() 将始终执行**重新渲染**操作。如果可变对象被使用，且无法在 shouldComponentUpdate() 中实现条件渲染，那么仅在新旧状态不一时调用 setState()可以避免不必要的重新渲染

**参数一为带有形式参数的 updater 函数**：
```javascript
(state, props) => stateChange
```
state 是对应用变化时组件状态的引用。当然，它不应直接被修改。你应该使用基于 state 和 props 构建的新对象来表示变化。例如，假设我们想根据 props.step 来增加 state：
```javascript
this.setState((state, props) => {
  return {counter: state.counter + props.step};
});
```
updater 函数中接收的 state 和 props 都保证为**最新**。updater 的**返回值**会与 state 进行`浅合并`。

setState() 的第二个参数为可选的**回调函数**，它将在 setState 完成合并并重新渲染组件后执行。通常，我们建议使用 `componentDidUpdate()` 来代替此方式。

setState() 的第一个参数除了接受函数外，还可以接受对象类型：
```javascript
setState(stateChange[, callback])
```
stateChange 会将传入的对象浅层合并到新的 state 中，例如，调整购物车商品数：

这种形式的 setState() 也是异步的，并且在同一周期内会对多个 setState 进行批处理。例如，如果在同一周期内多次设置商品数量增加，则相当于：
```javascript
Object.assign(
  previousState,
  {quantity: state.quantity + 1},
  {quantity: state.quantity + 1},
  ...
)
```
后调用的 setState() 将覆盖同一周期内先调用 setState 的值，因此商品数**仅增加一次**。如果后续状态取决于当前状态，我们建议使用 updater 函数的形式代替：
```javascript
this.setState((state) => {
  return {quantity: state.quantity + 1};
});
```

### forceUpdate(callback)

默认情况下，当组件的 state 或 props 发生变化时，组件将重新渲染。如果 render() 方法依赖于其他数据，则可以调用 `forceUpdate()` 强制让组件重新渲染。

调用 `forceUpdate()` 将致使组件调用 `render()` 方法，此操作会跳过该组件的 `shouldComponentUpdate()`。但其**子组件**会触发正常的生命周期方法，包括 `shouldComponentUpdate()` 方法。如果标记发生变化，React 仍将只更新 DOM。

通常你应该**避免使用** forceUpdate()，尽量在 render() 中使用 this.props 和 this.state。
