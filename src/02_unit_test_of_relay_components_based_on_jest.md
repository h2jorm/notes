# Relay组件基于jest的单元测试
*leeching 2015-12-27*

*文章中采用了此处提出的测试方案：[stackoverflow链接](http://stackoverflow.com/a/32908821)*

*不涉及[Relay](http://facebook.github.io/relay)和[jest](http://facebook.github.io/jest)的基础用法*

简单地说，Relay组件是附着在React组件外层的一层包装，负责统筹调度整个React app和后台graphql服务器的数据通信工作。一个简单的Relay组件如下：

```js
import React from 'react'
import Relay from 'react-relay'
class Hello extends React.Component {
  render() {
    return (
      <div>Hello, {this.props.user.name}</div>
    )
  }
}
Hello = Relay.createContainer(Hello, {
  fragments: {
    user: () => Relay.QL`
      fragment on User {
        name
      }
    `
  }
})
```

在上面的代码中，由`Relay.createContainer`方法生成的新的`Hello`组件，其实还是一个货真价实的React组件。Relay通过这个container层，向原来的`Hello`组件注入了`user`数据。

## Relay测试的主要问题
虽然，Relay组件其实也还是React组件，但是，经过实际操作发现，和React组件相比，测试Relay组件存在一些困难。主要有以下一些难点：

1.  在测试框架层面，jest对relay的支持度不够。比如，jest声称提供将异步代码进行同步测试的能力，但是对于在`Relay.RootContainer`组件中申明的异步数据请求，在测试代码中的渲染组件却无法做到同步渲染。这一功能的缺失直接导致了`Relay.RootContainer`组件无法被测试。
2.  在Relay层面，整个数据流都被封装起来了，没有对外暴露的足够的api用于探查当前的数据状态。对于普通开发者来说，Relay就是一个黑盒子。在编写业务代码时，这并不构成问题，但在单元测试中，无法有效探查组件内部的数据状态是致命的缺陷。

## Relay组件中的单元测试覆盖边界
单元测试有其[覆盖边界](src/unit_test_of_front_end.md#单元测试的覆盖边界)，边界之外的内容不应该被测试。

`Relay.RootContainer`组件申明的异步数据请求这个动作是在单元测试的覆盖边界之外的，所以，这一部分是不应该被测试的。应该始终假定，组件的数据请求都会获得预期的数据，并且直接将这些数据注入到下一级组件，比如`Relay.createContainer(ReactComponent)`中进行测试。

以下是一个简单的`Relay.RootContainer`
```js
import React from 'react'
import Relay from 'react-relay'
//`Hello`来自上一段代码
import Hello from './hello'

class UserRoot extends Relay.Route {
  static routeName = 'User'
  static queries = {
    user: Component => Relay.QL`
      query {
        user(id: $userId) {
          ${Hello.getFragment('user')}
        }
      }
    `
  }
}

class App extends React.Component {
  render() {
    const route = new UserRoot({userId: '4008123123'})
    return (
      <Relay.RootContainer
        Component={Hello}
        route={route}
      />
    )
  }
}
```

## Relay的Mock方案
第二个问题，即Relay封装过于严密，内部数据无法被探查的解决方案，需要借助jest框架提供的[manual mock](http://facebook.github.io/jest/docs/manual-mocks.html#content)功能。这个方案在[stackoverflow](http://stackoverflow.com/a/32908821)上有被提到。

概括地说，就是重写Relay的一些方法，并在这些方法内部埋下一些钩子。在测试中借助这些钩子来探查Relay的内部状态。以下是mock代码
```js
var Relay = require.requireActual('react-relay')
var React = require('react')
module.exports = {
  QL: Relay.QL,
  Mutation: Relay.Mutation,
  Route: Relay.Route,
  Store: {
    update: jest.genMockFn()
  },
  createContainer: (component, containerSpec) => {
    const fragments = containerSpec.fragments || {}
    Object.assign(component, { getFragment: (fragmentName) => fragments[fragmentName] })
    return component
  }
}
```
这段代码的最主要是复写了`Relay.Store.update`，在Relay应用中，所有主动的数据变动都需要调用这个方法实现，现在，在这里埋下一个钩子之后，所有由组件发起的数据变动都能在测试中被探查到。

另一段代码如下
```js
const React = require('react')
const ReactDOM = require('react-dom')
const {renderIntoDocument} = require('react-addons-test-utils')
const RelayTestUtils = {
  renderContainerIntoDocument(containerElement, relayOptions) {
    relayOptions = relayOptions || {}
    const relaySpec = {
      forceFetch: jest.genMockFn(),
      getPendingTransactions: jest.genMockFn().mockImplementation(() => relayOptions.pendingTransactions),
      hasOptimisticUpdate: jest.genMockFn().mockImplementation(() => relayOptions.hasOptimisticUpdate),
      route: relayOptions.route || { name: 'MockRoute', path: '/mock' },
      setVariables: jest.genMockFn(),
      variables: relayOptions.variables || {}
    }
    return renderIntoDocument(
      React.cloneElement(containerElement, { relay: relaySpec })
    )
  }
}
module.exports = RelayTestUtils
```
这段代码主要复写了被`Relay.createContainer`方法包装后的React组件的`props.relay`对象，同样地在里面埋下了一些钩子。具体略。

## Relay测试的其他问题
### jest preprocessor
非常不建议使用`babel-jest`插件(即使这个插件是由babel官方维护的)，而是手写一个`preprocessor`，这样可以从容应对某些情况，比如在组件中`require('./style.less')`文件，会导致的语法错误问题。

链接：一份简单的[preprocessor.js](https://github.com/leeching/relay-test-demo/blob/master/scripts/babel-jest.js)

### jest测试代码中的源代码引入方式
在jest的测试代码中，似乎只能使用`require`来引入被测试的代码，原因不明。[官方文档](http://facebook.github.io/jest/docs/tutorial-react.html#content)中也使用了`require`
而在使用`require`引入的过程中，babel编译的`export default`语句会出现如下问题
```
// hello.js
export default let hello = 'world'

// main.js
require('./hello') === 'world' //false
require('./hello').default === 'world' //true
```
这个问题的最彻底的解决方案是，在源码中使用`module.export`来替代`export default`。
