---
title: Think in React - 01 如何设计组件？
date: 2018-06-11 13:48:27
tags: react
categories: 技术分享
---

Think in React 这一系列文章旨在解决团队整体技术栈迁移至 React 系中我们所面临的问题以及解决之道。
作为首篇，就让我们单刀直入，聊一聊在业务开发中可能最让人困惑的一个课题：组件设计。

## 组件

我们拆分代码至更小的模块以达到分治的目的，而组件本身就是对我们这种拆分的抽象，同时，组件帮助我们遵循 DRY 原则，达到代码的复用。
在 vue 中，我们有非常多的途径可以达到复用的目的， 比如，我们使用 directive 封装 dom 操作，用 mixin 封装组件间相同的逻辑
在 React 中，虽然语法和设计不尽相同，但不变的是 UI 的本质，<b>展示交互</b> 和 <b>数据流转</b>
那么由此我们先简单的把组件分为 2 大类别：<b>展示型组件</b> 和 <b>容器型组件</b>

## 展示型组件

我们通常把拥有如下特性的组件称之为展示型组件：

- 描述外观
- 通常有 DOM 结构和样式，可接收 props.children 作为被包含的部分
- 不依赖于应用的其他部分，如 Redux
- 与数据如何加载变化无关
- 通过 props 传入数据和回调函数
- 尽量不含有 state （即便有，也与 UI 相关）

展示型组件无论是概念还是实现都相对比较简单，你可能会想到 RN 原生的组件库里你常用的那些组件 View, Text，TouchableWithoutFeedback 等，他们大多只是承载组件的渲染，自身与数据无关，通过传入的 props 改变外观或相应用户的操作。
你也可以自由的在组件嵌套结构里插入组合别的自定义组件

```jsx
const styles = StyleSheet.create({
  page: {
    backgroundColor: '#fff',
    padding: 20
  }
})

// 简单的functional component
const PageFunctionalComponent = props => <View style={styles.page}>{props.children}</View>

// 或者你更倾向于 class
class PageClassComponent extends React.PureComponent {
  render() {
    return <View style={styles.page}>{this.props.children}</View>
  }
}
```

## 容器型组件

我们通常把拥有如下特性的组件称之为容器型组件：

- 描述如何运作
- <b>没有 DOM 结构(除了包裹用的节点)和样式</b>
- 不依赖于应用的其他部分，如 Redux
- 自身提供数据及表现函数
- 连接 redux 等并向内传递数据和 action
- 通常都是有状态的，作为数据源存在
- 通常通过 HOC 生成，或使用 render props

通过这些定义，大家可以发现我们的页面组件（继承自 Page）就是一个典型的容器型组件，他通过 HOC 包装，连接 Redux Store，为内部组件提供数据和 actions

### HOC 高阶组件

简单来说，HOC（high order component）就是一个接收 component 为入参，并返回一个新的 component 的函数
HOC 被用来替代 createClass 时期的 mixin [Mixins Considered Harmful](https://reactjs.org/blog/2016/07/13/mixins-considered-harmful.html)。解决了组件间复用公共数据逻辑的问题。
先来看一个简单的 withUser HOC 的实现，他提供了 user 对象作为数据传递给被包装的组件

```js
import hoistNonReactStatic from 'hoist-non-react-statics'
import { Page } from './components' // 一个展示型组件

const withUser = user => Component => {
  class ComponentWithUser extends React.Component {
    static displayName = `withUser(${Component.displayName || Component.name})`

    state = {
      user: user || { name: 'lily', logout() {}, getLocation() {} }
    }

    render() {
      return <Component {...this.props} user={this.state.user} />
    }
  }
  hoistNonReactStatic(ComponentWithUser, Component)
  return ComponentWithUser
}

const UserPage = withUser(Page) // 漂亮，你得到了一个容器型组件
```

注意点：

- 需要为组件设定静态属性 displayName，这样有助于 debug 时更好的定位到此 HOC 组件
- 不要忘记将 HOC 组件接收到的 props 传递给被包裹的组件
- 被包裹的组件的静态属性和方法需要使用 hoistNonReactStatic 提升至 HOC 组件（这也是为何不推荐继承而是使用组合）
- 需要额外接收参数请使用柯里化，这样符合 Rambda 演算 Unary 的特性，且可以转换为修饰器函数

### render props

起初它被大量运用在 chenlou 的 react-motion 库中，现在他正式出现在 React 官方文档的高级范式中。
他允许组件接收一个或多个函数类型的 props（返回值为组件），可以在容器 render 时，动态的将数据传递给这个函数，由此完成数据传递。
一般我们会使用 props.children 作为 render props。

```jsx
import { Page } from './components' // 一个展示型组件

class UserContainer extends React.Component {
  static propTypes = {
    children: PropTypes.func.isRequred
  }

  state = {
    user: { name: 'lily', logout() {}, getLocation() {} }
  }

  render() {
    return <React.Fragment>{this.props.children(this.state.user)}</React.Fragment>
  }
}

class UserPage extends React.Component {
  renderPage = user => <Page user={user} />

  render() {
    return <UserContainer>{this.renderPage}</UserContainer>
  }
}
```

注意点：

- 由于 render props 不寻常的设计，需要为组件声明 propTypes 的 children 属性为 PropTypes.func.isRequred，以避免错误的使用
- 不要使用行内的函数声明，虽然他很简单，但在组件 rerender 时，会重新生成一个函数，造成资源浪费，可以把函数转换成组件的方法
- React.Fragment 为占位标记，不会生成任何结构，符合容器型组件尽量不含 dom 结构的目的

### 两者比较

- 当使用 HOC 修饰 class 组件时，可以使用 es7 的修饰器语法简化代码

```js
@withUser({ name: 'lily' })
class Page extends React.Component {
  //...
}
```

- 多个 HOC 同时作用于一个组建时，可以被组合（compose），得益于函数式编程的特性
- HOC 可以和函数组合一样，组合 N 个 hoc 函数至同一组件，但这样也会带来许多问题诸如 静态属性 的提升，props 的重名问题，同事也会造成 debug 的调用栈过深，以及不知道 props 是从哪一个 HOC 传入的。<b>所以需要谨慎设计你的 HOC</b>
- 设计 HOC 函数时需要大量的模板代码以支撑上述论点
- render props 的特性使得他能够够好的支撑运行时，而不像 HOC 那样需要静态组合
- 我的个人建议：HOC 可以作为顶层的数据和方法传入，而 render props 则适合局部的数据传递

### 案例

- Container with restful resource

```js
const withService = ({ url, params }) => BaseComponent => {
  class ComponentWithService extends React.Component {
    constructor(props) {
      this.initialSnapshot = { params }
      this.state = {
        data: null,
        params
      }
    }
    state = {
      data: null
    }
    componentDidMount() {
      this.fetchData()
    }

    componentDidUpdate(prevProps, prevState) {
      if (!_.isEqual(prevState.params, this.state.params)) {
        this.fetchData()
      }
    }

    fetchData = async () => {
      const data = await client.post(url, this.state.params)
      this.setState({ data })
    }

    setParams = newParams => {
      this.setState({
        params: { ...this.state.params, ...newParams }
      })
    }

    refresh = () => {
      this.setState({
        params: this.initialSnapshot.params
      })
    }

    render() {
      const serviceProps = { state: this.state, setParams: this.setParams, refresh: this.refresh }
      return <BaseComponent {...this.props} service={serviceProps} />
    }
  }

  return ComponentWithService
}

// 使用

const Page = props => <Page />

withService({
  url: '/api',
  params: {
    a: 1
  }
})(Page)

// or

@withService({
  url: '/api',
  params: {
    a: 1
  }
})
class Page extends Component {
  //...
}
```

### 好用的第三方库

- [recompose](https://github.com/acdlite/recompose) 提供了一系列相当实用的 HOC 方法

### 其他实现

当然出于 react props 的自由度，还有许多可作为实现组件复用的方法，比如 Component Inject, 或者 Nested Component。但作为一个团队的开发范式和最佳实践，以及 React 官方的推荐，决定使用上面提到的 2 种范式对组件进行复用。

## 总结

当我们完成这样的代码切割后，我们最终获得了什么？

- 我们的展示型组件将非常容易测试，因为他们不包含状态，只需要传入相应的 props，就可以验证展示的逻辑是否符合预期。今后的章节会介绍如何测试自己的组件，以提高组件的健壮性
- 当我们熟练运用容器化组件，我们会很容易提取出行为层和数据层，使我们的代码分层更准确更清晰

参考资料：

- [Presentational and Container Components](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0)
- [A tale of Higher-Order Components & Render Props](https://medium.com/ingenious/a-tale-of-higher-order-components-render-props-a1ba47e8cfeb)
