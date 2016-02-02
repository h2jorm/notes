# React组件基于karma+jasmine的单元测试
*leeching 2016-02-01*

- [前端单元测试难点和解决方案](#前端单元测试难点和解决方案)
- [jasmine VS jest](#jasmine-vs-jest)
- [测试的基础步骤](#测试的基础步骤)
  - [准备组件](#准备组件)
  - [引入组件](#引入组件)
  - [mock数据](#mock数据)
  - [render组件](#render组件)
  - [执行测试用例](#执行测试用例)
  - [清理](#清理)
- [测试的基本内容](#测试的基本内容)
- [mock](#mock)
  - [第三方库](#第三方库)
  - [异步请求](#异步请求)
- [高阶组件的测试方案](#高阶组件的测试方案)
  - [react-redux组件](#react-redux组件)

本文将详细介绍如何在karma+jasmine技术框架下对react组件进行单元测试。

## 前端单元测试难点和解决方案
简单的说，前端代码测试的难点主要在于两方面。

一是，代码模块化做得不够好。目前，主流的前端模块化方案有三种，分别是被nodeJS采用的CommonJS方案，es2015规范提出的`import`、`export`语法，和AMD方案。另外，目前react开发大多采用`webpack`+`babel`的建构方案，在这种建构方案的基础上，一般会倾向于采用es2015的模块化方案加上少量的`module.exports`、`exports`语法。

相比在`jQuery`时代普遍采用闭包来实现作用域隔离，react的模块化方案显然更优。但是，前端代码长久以来的风格陋习还是比较严重。即使是使用模块化的开发开发，模块之间随意调用，模块的层级关系不够明确，增加了代码的耦合度。

另外一个难点是，前端代码中存在大量的异步逻辑。前端代码中的异步主要有三种。一种是dom元素上的回调事件。二是ajax异步通信请求。三是es2015中原生支持的`Promise`语法。`jasmine`提供了`done`方法用于支持异步代码的测试，但是，只依赖于测试框架来解决异步代码的测试问题是不够的。

对于react开发来说，在`ReactElement`上申明的事件回调实际上在虚拟dom中是同步运行的。所以在单元测试中，借助这个特性可以规避掉所有由回调事件产生的异步难题。至于剩余两种异步的解决思路是，尽可能将异步逻辑简化并集中一处。将这些异步逻辑通过mock的方式在单元测试中规避掉。具体的mock方案在下文。

## jasmine VS jest
jest是在jasmine的基础上开发的一套测试框架，在react的开源生态中使用比较广泛。但是，在实际使用中发现了一些问题：

- 配置jest比较麻烦
  1. babel-jest.js（es2015的预处理）
  2. package.json（jest的配置）
- jest测试运行中的报错信息难以解读
- jest的单元测试中的console无法在terminal中显示，不方便调试
- jest会自动mock所有包，实际上，在测试react组件时，大部分包并不需要mock

当然，jest也是有优点

- `__tests__`、`__mocks__`的命名方案可以借鉴
- mock机制很好用

通过探索，个人认为karma+jasmine组合方案的实用性要优于jest。`webpack`官方提供了`karma-webpack`插件，用于预处理es2015代码。可以使用`karma-sourcemap`插件生成sourcemap，方便调试。因为`karma-webpack`提供了配置`webpack`的可能性，jest的mock机制可以通过`webpack`的`resolve.modulesDirectories`api非常简单地模拟出来。以下是一个简单的例子，完成了对`react-redux`的简单mock。

```js
// karma.conf.js
const path = require('path')
module.exports = function(config) {
  config.set({
    basePath: '',
    frameworks: ['jasmine'],
    files: [
      'src/**/__tests__/**/*.spec.js',
      'src/**/__tests__/*.spec.js'
    ],
    exclude: [
    ],
    preprocessors: {
      'src/**/__tests__/**/*.spec.js': ['webpack', 'sourcemap'],
      'src/**/__tests__/*.spec.js': ['webpack', 'sourcemap'],
    },
    webpack: {
      devtool: 'inline-source-map',
      resolve: {
        modulesDirectories: [
          path.resolve('__mocks__'),
          path.resolve('node_modules')
        ],
        alias: {
          '#': path.resolve('src')
        }
      },
      module: {
        loaders: [
          {
            test: /\.js$/,
            loader: 'babel',
            include: /src|__mocks__/,
            query: {
              presets: ['react', 'es2015', 'stage-0'],
            }
          },
          {
            test: /\.less$/,
            loader: 'style!css!less'
          },
          {
            test: /\.(png|jpg|gif)$/,
            loader: 'url',
            query: {
              limit: 2048,
            }
          },
        ]
      }
    },
    webpackMiddleware: {
      noInfo: true
    },
    reporters: ['progress'],
    port: 9876,
    colors: true,
    logLevel: config.LOG_INFO,
    browsers: ['Chrome'],
    singleRun: true
  })
}

// __mocks__/react-redux.js
import {
  connect as _connect,
  Provider,
} from '../node_modules/react-redux';

const connect = function (mapStateToProps, mapDispatchToProps, mergeProps, options = {}) {
  Object.assign(options, {withRef: true});
  return _connect(mapStateToProps, mapDispatchToProps, mergeProps, options);
};

module.exports = {
  connect,
  Provider,
};
```

## 测试的基础步骤

### 准备组件
判断待测试的组件是否能够被测试。和外部模块耦合度过高的组件很难被测试。

此外，还应该判断该组件是否有被测试的价值，逻辑过于简单和逻辑过于复杂的组件都没有测试的价值。前者可以在测试其父组件的同时顺便测一下，后者应该在模块内部划分成更小的组件以提高测试效率并降低测试难度。

### 引入组件
采用jest的方案，在靠近组件的位置建立`__tests__`文件夹，在该文件夹下建立和组件的同名测试文件，并以`.spec.js`作为后缀名。在该文件中引入组件，以及和测试组件相关的模块，比如`reactDOM`、`react-addons-test-utils`等

### mock数据
mock数据主要有两部分，一是组件的props，二是异步函数。mock的工作主要在`beforeEach`这个钩子函数中进行。具体见示例。

### render组件
调用`react-addons-test-utils`提供的`renderIntoDocument`api渲染待测试组件。一般测试中需要获取到这个组件渲染后的实例以及组件的真实dom。建议编写一个helper函数来简化这个常用操作。

```js
// src/utils/testHelper.js
import {renderIntoDocument} from 'react-addons-test-utils';
export function renderComponent(ReactElement) {
  const instance = renderIntoDocument(
    ReactElement
  );
  const node = ReactDOM.findDOMNode(instance);
  return {instance, node};
}
```

### 执行测试用例
根据需要编写测试用例，测试用例在jasmine中会按顺序运行。

### 清理
jasmine提供了`afterEach`api，用于在执行完每个测试用例之后做清理工作。在测试react组件完毕或，会经常需要调用`ReactDOM.unmountComponentAtNode`api来卸载在`beforeEach`中渲染到dom的组件。如果测试用例中操作了`redux`的store状态，也应该在`afterEach`回调中将store的状态重置。

```js
// src/components/Hello.js
import React from 'react';

export default class Hello extends React.Component {
  constructor() {
    super();
    this.state = {
      clicked: false
    };
  }
  clickHandler() {
    this.setState({
      clicked: !this.state.clicked
    });
  }
  render() {
    return (
      <div>Hello, {this.props.name}</div>
    );
  }
}
// src/components/__tests__/Hello.spec.js
import React from 'react';
import ReactDOM from 'react-dom';
import {Simulate} from 'react-addons-test-utils';
import {renderComponent} from '#/utils/testHelper';// 见上文
import Hello from '../Hello';

describe('Hello', () => {
  let hello, name, mock;
  beforeEach(() => {
    name = 'world';
    hello = renderComponent(
      <Hello name={name} />
    );
  });
  afterEach(() => {
    ReactDOM.unmountComponentAtNode(hello.dom.parentNode);
  });
  it('should set initial state `clicked` false', () => {
    expect(hello.instance.state.clicked).toBeFalsy();
  });
  it('should render `name`', () => {
    expect(hello.node.textContent).toBe(`Hello, ${name}`);
  });
  it('should toggle state `clicked` when clicking the component', () => {
    Simulate.click(Hello);
    expect(hello.instance.state.clicked).toBeTruthy();
    Simulate.click(Hello);
    expect(hello.instance.state.clicked).toBeFalsy();
  });
});
```

## 测试的基本内容
测试的内容以具体项目需要做适当增减。一般来说，需要覆盖以下内容。

- 各阶段的数据渲染是否准确
- 各阶段的组件状态是否准确
- 回调执行后的数据和状态变动是否符合预期
- api请求的地址，类型和发送数据是否符合预期
- helper函数的逻辑是否符合预期

组件中有些部分是不需要测试的

- 和组件状态无关的样式不测
- 如果组件内部使用了外部库提供的组件，默认假定这些组件都会正常运行。只需要测试向这些组件提供的props数据是否准确，不需要测试这些组件的运行结果。

## mock

### 第三方库
使用像`react-router`之类的库会向组件传递`route`、`routeParams`之类的`props`。如果组件使用到了这些`props`，那么在单元测试时，渲染这些组件的时候，应该mock相应的`props`，并传给组件。

### 异步请求
方便起见，将所有ajax请求都统称为`fetch`。

`fetch`是所有前端系统中都会被频繁调用的异步函数。为了方便在单元测试中做mock，建议根据项目的实际情况将`fetch`逻辑做简化抽象，并封装成一个统一的service。所有的异步请求都调用这个service。

```js
// src/utils/fetch.js
import fetch from 'isomorphic-fetch';
let myFetch = fetch;
exports.config = function (fn) {
  myFetch = fn;
};
exports.fetch = function (method, url, data) {
  let config = {
    method,
    headers: {
      'Accept': 'application/json',
      'Content-Type': 'application/json',
    },
  };
  if (data)
    Object.assign(config, {
      body: JSON.stringify(data)
    });
  return myFetch(url, config).then(res => {
    return res.json();
  });
};

// src/components/List.js
import React from 'react';
import {fetch} from '#/utils/fetch';
export default class List extends React.Component {
  componentDidMount() {
    fetch('GET', '/api/list')
    .then(data => {
      this.setState({
        list: data
      });
    });
  }
  render() {
    return (
      <ul>
        {
          this.state.list.map(item =>
            <li>{item.name}</li>
          )
        }
      </ul>
    );
  }
}
// src/components/__tests__/List.spec.js
import ReactDOM from 'react-dom';
import {Simulate} from 'react-addons-test-utils';
import {config} from '#/utils/fetch';
import {renderComponent} from '#/utils/testHelper';// 见上文
import List from '../List';

describe('List', () => {
  let list, name, mock;
  beforeEach(() => {
    name = 'world';
    mock = {
      fetch: function () {
        return Promise.resolve();
      }
    };
    spyOn(mock, 'fetch').and.callThrough();
    list = renderComponent(
      <List />
    );
  });
  afterEach(() => {
    ReactDOM.unmountComponentAtNode(list.dom.parentNode);
  });
  it('should launch a request', () => {
    expect(mock.fetch.calls.count()).toBe(1);
    expect(mock.fetch.calls.argsFor(0)[0]).toBe('GET');
    expect(mock.fetch.calls.argsFor(0)[1]).toBe('/api/list');
  });
});
```

## 高阶组件的测试方案
高阶组件指的是，将作为ui的react组件作为子组件包裹起来，并通过`props`向其注入数据的react组件。一般高阶组件不涉及ui样式，只负责向子组件注入数据。在react中，常见的高阶组件有，调用`react-redux`的`connect`api生成的组件，调用`Relay`的`createContainer`api生成的组件。

高阶组件在实质上还是react组件，所以原则上上述的测试说明都是通用的。但是，测试高阶组件有一些难度。首先是如何获取到子组件的渲染实例和真实dom的问题。事实上，像`react-redux`和`react-relay`这样的框架或库，如果没有设计相关的api的话，是很难获取到子组件的。

在`react-redux`的`connect`方法中，如果传入第四个参数`{withRef: true}`，那么在生成的高阶组件的实例上调用`getWrappedInstance`方法，会返回子组件的实例。上面已经给出`react-redux`的[mock代码](#jasmine VS jest)。

但是，`react-relay`并没有给出类似的api，虽然可以通过一些手段来mock`react-relay`的api，在其中埋下一些钩子函数用来探查子组件的状态。但是总体来看，测试`relay`组件的代价是比较高的。

### react-redux组件
当前使用`redux`来管理`react`的系统状态是比较流行的技术解决方案，但`redux`本身是独立运行的一个单例，需要通过`react-redux`之类的中间库，将存放在`store`内部的状态数据引入到react组件的单向数据流中。也就是调用`connect`api向少数smart组件注入数据，从而形成高阶组件。

对于这种应用，单元测试的方案有两种。一种是不测试这些高阶组件，采用mock`props`的方式，直接测试作为子组件的react组件。独立测试`redux`。另一种是，直接测试高阶组件。两者都是可行的，各有利弊，可酌情考虑。前一种测试方案比较直接明了不具体说明。这里给出第二种测试方案的执行细节。

```js
// __mocks__/react-redux.js
// ...
// 具体代码见上文
// ...

// src/utils/testHelper.js
// 是上文中的同一个`testHelper`文件
export function renderConnectComponent(ReactElement) {
  // `renderComponent`方法见上文
  let {instance} = renderComponent(ReactElement);
  instance = instance.refs.main.getWrappedInstance();
  const node = ReactDOM.findDOMNode(instance);
  return {instance, node};
}

// src/components/Hello.js
import React from 'react';
import {connect} from 'react-redux';
class Hello extends React.Component {
  render() {
    return <div>Hello, {this.props.name}</div>
  }
}
export default connect(state => ({
  name: state.user.name
}))(Hello);

// src/components/__tests__/Hello.spec.js
import React from 'react';
import ReactDOM from 'react-dom';
import {Provider} from 'react-redux';
import {renderConnectComponent} from '#/utils/testHelper';
import store from '#/store';
import Hello from '../Hello';

describe('Hello', () => {
  let hello;
  beforeEach(() => {
    class HelloProvider extends React.Component {
      render() {
        return (
          <Provider store={store}>
            <Hello ref="main"/>
          </Provider>
        );
      }
    }
    hello = renderConnectComponent(
      <HelloProvider />
    )
  });
  afterEach(() => {
    // 重置store
    store.dispatch({type: 'RESET'});
  });
  it('should....', () => {
    // 编写单元测试
  });
});
```
