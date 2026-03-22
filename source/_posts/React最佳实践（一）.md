---
title: React最佳实践（一）
date: 2020-08-14 17:02:16
tags:
  - React
categories:
  - React
---

## 条件语句

JSX 有许多不同方式来表达条 件逻辑，理解每种方式的益处及其存在的问题对于编写可读且可维护的代码非常重要。

1. 显示一个注销按钮时

```javascript
// 下述做法可行，但可读性不好，组件合条件很多时会更差
let button;
if (isLoggedIn) {
  button = <LogoutButton />;
}
return <div>{button}</div>;
```

优化

```javascript
// JSX 可以利用行内条件来判断：
<div>{isLoggedIn && <LoginButton />}</div>
```

2. if...else 语句

```javascript
let button;
if (isLoggedIn) {
  button = <LogoutButton />;
} else {
  button = <LoginButton />;
}
return <div>{button}</div>;
```

优化

```javascript
<div>{isLoggedIn ? <LogoutButton /> : <LoginButton />}</div>
```

3. 来看更复杂的情况

```javascript
// 需要多个变量才能判断是否要渲染组件：
<div>{dataIsReady && (isAdmin || userHasPermissions) && <SecretData />}</div>
```

优化

```javascript
// 上述示例中的行内条件语句的写法很好，
// 但可读性受到了很大影响。此时可以在组件内编写 一个辅助函数来检验 JSX 的条件语句:
canShowSecretData() {
    const { dataIsReady, isAdmin, userHasPermissions } = this.props
    return dataIsReady && (isAdmin || userHasPermissions)
}
<div>
{this.canShowSecretData() && <SecretData />}
</div>
```

在优化（使用 getter 方法来取代函数）

```javascript
get canShowSecretData() {
    const { dataIsReady, isAdmin, userHasPermissions } = this.props
    return dataIsReady && (isAdmin || userHasPermissions)
    }
<div>
{this.canShowSecretData && <SecretData />}
</div>
```

## 函数式编程基础

### 纯粹性

```javascript
// 纯粹的函数
const add = (x, y) => x + y;

// 非纯粹的函数
let x = 0;
const add = (y) => (x = x + y);
```

### 不可变性

```javascript
// 以下代码不遵循不可变性
const add3 = (arr) => arr.push(3);
const myArr = [1, 2];
add3(myArr); // [1, 2, 3]
add3(myArr); // [1, 2, 3, 3]

// 可以改用concat满足不可变性
const add3 = (arr) => arr.concat(3);
const myArr = [1, 2];
const result1 = add3(myArr); // [1, 2, 3]
const result2 = add3(myArr); // [1, 2, 3]
```

### 柯里化

```javascript
// 原先的写法
const add = (x, y) => x + y;

// 改成柯里化:
// 这种函数写法相当方便，因为传入第一个参数后，
// 第一个值被保留起来，返回的第二个函数 可以多次复用。
const add = (x) => (y) => x + y;
```

### 组合

```javascript
const add = (x, y) => x + y;
const square = (x) => x * x;

const addAndSquare = (x, y) => square(add(x, y));
```

## 开发可复用的组件

### 使用 ES2015 语法类来创建组件

```javascript
// 不推荐
const Button = React.createClass({ render() {
    return <button />
})

// 推荐使用继承React.Component
class Button extends React.Component {
    render() {
        return <button />
    }
}
```

- 主要区别：
  - prop
  - 状态
  - 自动绑定

### 自动绑定

```javascript
// 以下的简单示例
const Button = React.createClass({
  handleClick() {
    console.log(this);
  },
  render() {
    return <button onClick={this.handleClick} />;
  },
});

// 如果使用继承的方法（this结果会是null）
class Button extends React.Component {
  handleClick() {
    console.log(this);
  }
  render() {
    return <button onClick={this.handleClick} />;
  }
}

// 解决：
// 1）箭头函数：自动将当前的 this 绑定到函数体
class Button extends React.Component {
  handleClick() {
    console.log(this);
  }
  render() {
    return <button onClick={() => this.handleClick()} />;
  }
}
```

<font color='red'>注意：</font> 这样做符合预期，也不会带来什么特殊问题。唯一的缺点在于，如果在意性能，那么就需要理解代码的本质。  
实际上，在渲染方法中绑定函数会带来无法预料的副作用，因为每次渲染组件(应用在生命 周期内会多次渲染组件)时都会触发箭头函数。虽然在渲染方法内多次触发某个函数不太理想，但本身并没有什么问题。 <font color='red'>问题在于，如果这个函数传递给子组件，那么子组件在每次更新过程中都会接收新的 prop。</font>这可能会导致低效的渲染，进而引发问题，对于纯粹组件而言尤其如此。

解决函数绑定问题最佳方案是在构造器内进行绑定操作，这样即使多次渲染组件，它也不会发任何改变。

```javascript
class Button extends React.Component {
  constructor(props) {
    super(props);
    // 就是这样，问题解决了!
    this.handleClick = this.handleClick.bind(this);
  }
  handleClick() {
    console.log(this);
  }
  render() {
    return <button onClick={this.handleClick} />;
  }
}
```

### 无状态函数式组件

- React 0.14 引入了这个方法。它十分强大，可以使得代码更易维护和复用。
- 语法相当简洁优雅，查看以下示例:

```javascript
() => <button />;
```

1. props 与上下文

```javascript
// 无状态函数式组件可以接收 props 对 象作为参数:
props => <button>{props.text}</button>

// 更简洁的 ES2015 解构语法:
({ text }) => <button>{text}</button>

// 无状态函数就可以通过 propTypes 属性来接收 props
const Button = ({ text }) => <button>{text}</button>
Button.propTypes = {
  text: React.PropTypes.string,
}

// 无状态函数式组件也接收表示上下文的第二个参数。
(props, context) => (
  <button>
    {context.currency}
    {props.value}
  </button>
)
```

1. 关键词 this && 状态 && 生命周期
   无状态函数式组件与状态组件的一项区别在于，this 在无状态函数式组件的执行过程中不指向组件本身。  
   由于这个原因，与组件实例相关的 setState 等方法以及生命周期方法都无法使用。  
   <font color='red'>新版的 React 出了 hooks 之后这个问题将不存在</font>

2. ref 与事件处理器

```javascript
// 因为无状态函数式组件不能访问组件实例，
// 所以如果要使用 ref 或者事件处理器，需要按以下方式来定义。
() => {
  let input;
  const onClick = () => input.focus();
  return (
    <div>
      <input ref={(el) => (input = el)} />
      <button onClick={onClick}>Focus</button>
    </div>
  );
};
```

### 状态

- 某些情况下可能需要在状态更新完成时执行一些操作，React 为此提供了一个回调 函数:

```javascript
this.setState({ clicked: true }, () => {
  console.log("the state is now", this.state);
});
```

- 异步
  - 应该总是将 setState 方法当作异步的

```javascript
// 在事件处理器中触发了 setState 后，尝试将当前状态值打印到控制台中，那 么获得的是旧状态值:
handleClick() {
    this.setState({
        clicked: true,
    })
    console.log('the state is now', this.state) // the state is now null
}
render() {
    return <button onClick={this.handleClick}>Click me!</button>
}

// 稍微修改一下代码:
handleClick() {
    setTimeout(() => {
        this.setState({
            clicked: true,
        })
        console.log('the state is now', this.state)
        // 结果： the state is now Object {clicked: true}
    })
}
```

示例使用 setTimeout 只是为了展示 React 的行为，<font color='red'>你永远不要这样编写事件监听器</font>。

### 可派生的值

- 只要能根据 props 计算最终值，就不应该将任何数据保存在状态中。

```javascript
class Price extends React.Component {
  constructor(props) {
    super(props)
    this.state = {
      price: `${props.currency}${props.value}`, // 不推荐
    }
  }
  render() {
    return <div>{this.state.price}</div>
  }
}

// 如果在父组件中按照以下方式创建，那么这种做法是可行的:
<Price currency="£" value="100" />

// 问题在于，如果货币单位或价格在 Price 组件的生命周期内发生改变，
// 则永远不会重新计算状态(因为只会调用构造器一次)，应用就会显示错误的价格。

// 因此，只要可以，就应该用 props 来计算值：
getPrice() {
    return `${this.props.currency}${this.props.value}`
}
```

### prop 类型

- 用 prop 类型定义清晰的接口，以便组件可以复用。

### 可复用组件

## 组合一切

### 组件间的通信

- children

### 容器组件与表现组件模式

- React 组件通常包含杂合在一起的逻辑与表现。
- 逻辑：一般指与 UI 无关的那些东西，如 API 的调用、数据操作以及事件处理器。
- 表现：则是指渲染方法中创建元素用来显示 UI 的部分。
- 容器组件：
  - 更关心行为部分
  - 负责渲染对应的表现组件
  - 发起 API 请求并操作数据
  - 定义事件处理器
  - 写作类的形式
- 表现组件：
  - 更关心视觉表现
  - 负责渲染 HTML 标记
  - 以 props 的形式从父组件接收数据
  - 通常写作无状态函数组件

### mixin

### 高阶组件

### recompose

### 函数子组件

## 恰当地获取数据

### 数据流

- 单向数据流，每个组件都以 prop 的形式从父组件接收数据，并且 prop 无法修改。

#### 子组件与父组件的通行（回调函数）

#### 公有父组件

#### 数据获取

- 用于获取数据的代码可以放在两个生命周期钩子中:componentWillMount 和 componentDidMount
- 前者会在组件首次渲染前触发，后者则在组件挂载完成后立即触发。
- 使用前者似乎是正确的做法，毕竟我们希望尽快加载数据，不过需要注意一点。实际上，服务端渲染和客户端渲染都会触发 componentWillMount 函数。
- <font color='red'>因此，我们只能用 componentDidMount 钩子</font>

1. 基于高阶函数封装数据请求

```javascript
// 基于高阶函数封装请求
const withData = (url) => (Component) =>
  class extends React.Component {
    constructor(props) {
      super(props);
      this.state = { data: [] };
    }
    componentDidMount() {
      // 模拟虚拟请求
      setTimeout(() => {
        this.setState({
          data: [
            { id: "001", title: "小明", excerpt: "开发工程师" },
            { id: "002", title: "老王", excerpt: "设计师" },
          ],
        });
      }, 1000);
    }
    render() {
      return <Component {...this.props} {...this.state} />;
    }
  };

const withGists = withData("https://api.github.com/users/gaearon/gists");

const List = (props) => {
  console.log("List -> gists", props);
  return (
    <ul>
      {props.data.map((gist) => (
        <li key={gist.id}>{gist.title}</li>
      ))}
    </ul>
  );
};
List.propTypes = {
  data: PropTypes.array,
};

const ListWithGists = withGists(List);
```

2. 我们可以用 react-refetch 库来替换我们写的高阶函数

```javascript
import { connect as refetchConnect } from "react-refetch";

const List = ({ gists }) => {
  if (gists.pending) {
    return <div>loading...</div>;
  } else if (gists.rejected) {
    return <div>{gists.reason}</div>;
  } else if (gists.fulfilled) {
    return (
      gists.fulfilled && (
        <ul>
          {gists.value.map((gist) => (
            <li key={gist.id}>{gist.description}</li>
          ))}
        </ul>
      )
    );
  }
};

const ListWithGists = refetchConnect(() => ({
  gists: `https://api.github.com/users/gaearon/gists`,
}))(List);
```

## 表单&&事件

### 自由表单组件

### 受控表单组件

### 自动生成表单

### 事件

## 样式

### 行内样式

```javascript
const style = {
  color: "palevioletred",
  backgroundColor: "papayawhip",
};
const Button = () => <button style={style}>Click me!</button>;
```

### Radium（解决行内样式问题）

### CSS 模块

- 行内样式方案不适合项目与团队，但仍然希望尽量紧密结合样式与组件， 那么还有一个名为 CSS 模块的方案供你选择。
- Webpack && 项目搭建
- 局部作用域的 CSS
- 原子级 CSS 模块
- React CSS 模块
- Styled Component

## 服务端渲染

### 使用服务端渲染的原因

1. SEO
   - 在服务端渲染应用的一个主要原因就是 SEO。
2. 性能更强
   - 客户端渲染缺点：操作应用前，需要加载并运行文件包。
3. 不要低估复杂度
   - 维护通用应用
   - 要搭建和维护带有路由和逻辑的服务器、管理服务端数据流等
   - 缓存 服务器内容，以便更快地输出页面
   - ...
4. 如果发现加载整个应用耗时很久，而你已经采取了一切优化手段(下面将详细讨论优化这一话题)，那么可以考虑用服务端渲染提升感知速度，以便为用户提供更好的体验。

### Next.js

## 性能提升

### key 属性

- 安装 npm install --save-dev react-addons-perf 插件帮助记录性能相关信息

### 优化手段

1. 构建打包时要将 NODE_ENV 环境变量设置为 production
2. 压缩最终代码来减小体 积，以便应用可以更快加载（new webpack.optimize.UglifyJsPlugin()）

- 是否要更新组件（shouldComponentUpdate）
  - 如果比较后发现什么都没有改变，那么就不修改 DOM，这是理想情况。但如果渲染方法执 行了很复杂的操作，React 就要耗费一些时间才能判断出不需要进行任何修改，这种情况就不够 理想了。

```javascript
// 这个方法就是 shouldComponentUpdate，如果它返回 false，那么在父组件的更新过程中，组件及其全部子元素的渲染方法不会被调用。
shouldComponentUpdate() {
    return false
}
```

- 可以使用 React.PureComponent 做浅比较，这样不用维护 shouldComponentUpdate

3. why-did-you-update
4. 在渲染方法中创建函数
5. props 常量
6. 重构与良好设计
