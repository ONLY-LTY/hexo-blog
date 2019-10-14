---
title: React 学习笔记
date: 2016-12-19 18:04:09
tags:
  - React
---
学习官方文档的一些总结和笔记

### 一 基础语法
```html
<body>
    <div id="root"></div>
    <script type="text/babel">
      ReactDOM.render(
        <h1>Hello, world!</h1>,
        document.getElementById('root')
      );
    </script>
</body>
```
```html
<body>
    <div id="root"></div>
    <script type="text/babel">
      var names = ['react', 'javascript', 'html'];
      ReactDOM.render(
        <div>
        {
          names.map(function (name) {
            return <div>Hello, {name}!</div>
          })
        }
        </div>,
        document.getElementById('root')
      );
    </script>
</body>
```
上面就是简单的`JSX`语法。其实就是将`html`语法和和`JavaScript`语法混合在一起。解析器解析的时候遇见 `<` 开头的就用html规则解析，遇到代码块 `{}` 就用`JavaScript` 规则解析。`React` 官方推荐我们使用`JSX`语法，只是推荐，我们也可以使用纯的`JavaScript`代码写也是可以的，代码如下，只是使用`JSX`语法，组件的结构和组件的关系看上去比较清楚。使用`JSX`的时候需要指定`type`为`text/babel`，详细的`JSX`语法可以参考官方文档 `https://facebook.github.io/react/docs/introducing-jsx.html`

```html
<!--不使用JSX语法-->
<body>
    <div id = "root"/>
    <script type="text/babel">
    class Hello extends React.Component {
      render() {
        return React.createElement('div', null, `Hello ${this.props.toWhat}`);
      }
    }

    ReactDOM.render(
      React.createElement(Hello, {toWhat: 'World'}, null),
      document.getElementById('root')
    );
    </script>
</body>
```

### 二 Component
```html
<body>
    <div id="root"></div>
    <script type="text/babel">
     //通过React.createClass创建自定义组件
      var HelloWord = React.createClass({
        render: function() {
          //只能包含一个顶层标签
          return <h1>Hello {this.props.name}</h1>;
        }
      });
      ReactDOM.render(
        //把组件渲染到页面中
        <HelloWord name="Word" />,
        document.getElementById('root')
      );
    </script>
</body>
```
上面就是一个`React`简单的组件。`HelloWord`是一个组件类，<font color='red'>组件类的名称首字母必须大写</font>,自定义的组件必须通过`React.render`方法进行解析，在其他`html`标签中使用是没有效果的。`this.props` 是获取当前自定义组件的属性名称，获取对应的值就是加上对应的属性名称，比如`this.props.name` 。有一点例外就是 `this.props.children` 这个不是获取属性名称为children的值，它是获取组件里面的所有子节点对象。具体怎么使用可以查看官网的API `https://facebook.github.io/react/docs/react-api.html#react.children `我们使用组件属性的时候和普通的`html`标签没有什么区别。但是我们的属性可以指定任意类型。代码如下

```html
<body>
    <div id="root"></div>
    <script type="text/babel">
      var data = 1106;
      var CustomButton = React.createClass({
        //设置属性的类型
        propTypes: {
          title: React.PropTypes.string,
        },
        //设置属性的默认值
        getDefaultProps : function () {
          return {
            title : 'Hello World'
          };
        },
        render: function() {
          return <h1> {this.props.title} </h1>;
        }
      });

      ReactDOM.render(
        <CustomButton title={data} />,
        document.getElementById('root')
      );
    </script>
</body>
```
上面的代码我们给自己的组件`CustomButton`类定义了`propTypes`,指定属性名称`title`的类型为`string`，然后我们传入一个`int`类型的值给`title`，这样我们的`console`会提醒你 *Warning: Failed propType: Invalid prop `title` of type `number` supplied to `CustomButton`, expected `string`.*  这样就验证了属性的类型，在着我们可以指定`React.PropTypes.string.isRequired`来要求我们`title`这个属性是必须的，没有值的话会报错。以此来达到一些限制。我们制定了属性的类型之后，也可以设置属性的默认值，通过`getDefaultProps` 去设置。更多的具体属性类型可以看官方文档 `https://facebook.github.io/react/docs/react-api.html#typechecking-with-proptypes`

### 三 Component State

```html
<body>
    <div id="root"></div>
    <script type="text/babel">
    var CustomButton = React.createClass({
      //初始化myKey状态的值
      getInitialState: function() {
        return {myKey: false};
      },
      //定义组件的点击事件
      handleClick: function(event) {
        //设置状态liked的值
        this.setState({myKey: !this.state.myKey});
      },
      render: function() {
        var text = this.state.myKey ? 'myKey' : 'haven\'t myKey';
        return (
          <p onClick={this.handleClick}>
            You {text} this. Click to toggle.
          </p>
        );
      }
    });

    ReactDOM.render(
      <CustomButton />,
      document.getElementById('root')
    );
    </script>
</body>
```
上面的代码就是一个简单的组件状态的用法。在点击文本的时候设置不同的状态值进而渲染出不同的内容。只要执行`setState`的时候`React`会去重新计算`DOM`，渲染变化的值。除非在`shouldComponentUpdate`方法中返回`false`，个人觉得React最好的地方就是这个状态的转换，用户之间的交互更加流畅。<font color='red'>还有就是`setState`是异步的，`this.setState`会调用`render`方法，但并不会立即改变`state`的值，`state`是在render方法中赋值的。所以执行this.setState()后立即获取state的值是不变的。同样的直接赋值`state`并不会出发更新，因为没有调用render函数。</font>还有就是关于定义组件事件的，<font color='red'>React中绑定事件处理器是使用驼峰命名法的方式。</font>

### 四 Component Form
```html
<body>
    <div id="root"></div>
    <script type="text/babel">
    //自定义组件也可以继承React.Component
    class NameForm extends React.Component {
      constructor(props) {
        super(props);
        //构造方法初始化状态值
        this.state = {value: ''};
        //绑定事件的回调函数，这里是必须的。不然运行的时候会undefined
        this.handleChange = this.handleChange.bind(this);
        this.handleSubmit = this.handleSubmit.bind(this);
      }
      //定义组件事件
      handleChange(event) {
        this.setState({value: event.target.value});
      }
      //定义组件事件
      handleSubmit(event) {
        alert('A name was submitted: ' + this.state.value);
        event.preventDefault();
      }

      render() {
        return (
          <form onSubmit={this.handleSubmit}>
            <label>
              Name:
              <input type="text" value={this.state.value} onChange={this.handleChange} />
            </label>
            <input type="submit" value="Submit" />
          </form>
        );
      }
    }

    ReactDOM.render(
      <NameForm />,
      document.getElementById('root')
    );
    </script>
</body>
```
上面的代码是`React`处理表单，`React`认为表单的数据变化是用户和组件的一种交互，随着用户的输入，属性值是变化的，所以我们不能简单的用`this.props`去获取表单的值，因为`this.props`的值定义好了之后是不能变化的，而`React`做法是注册一个`onChange`事件的回调函数，通过实时的改变状态的值，进而获取和设置表单的值。而且 `textarea tags` 和 `select tag` 也是这种情况。具体可以参考官方文档 `https://facebook.github.io/react/docs/forms.html`

### 五 Component Lifecycle
```html
<body>
    <div id="root"></div>
    <script type="text/babel">
    var HelloLifecycle = React.createClass(
    {
          //初始化属性默认值,只调用一次，实例之间共享引用
          getDefaultProps:function() {
              console.log('getDefaultProps,1');
              return {
                  name1 : 'name1',
                  name2 : 'name2'
              };
          },
          // 初始化每个实例特有的状态
          getInitialState:function() {
              console.log('getInitialState,2');
              return null;
          },
          //组件渲染之前调用
          componentWillMount:function() {
              console.log('componentWillMount,3');
          },
          //只能访问this.props和this.state,只有一个顶层组件，不允许修改状态和DOM输出
          render: function() {
              console.log('component render 4');
              return <p>Hello ,{this.props.name1 + " " + this.props.name2}</p>;
          },
          //成功render并渲染完成真实DOM之后触发，可以修改DOM
          componentDidMount:function() {
              console.log('componentDidMount,5');
          },

          //运行中的函数
          //父组件修改属性触发，可以修改新属性、修改状态
          //obj为接收到修改了属性的对象，对象包含了修改后的属性在里边
          componentWillReceiveProps:function(obj) {
              console.log("运行中：1");
              console.log(obj);
          },
          //组件判断是否重新渲染时调用
          shouldComponentUpdate: function() {
              console.log("运行中：2");
              return true;
          },
          //组件更新前调用，不能修改属性和状态
          componentWillUpdate: function() {
              console.log("运行中：3");

          },
          //组件更新后调用
          //可以修改DOM
          componentDidUpdate:function() {
              console.log("运行中：5");
          },
          //销毁阶段，即把这个组件移出后执行的回调函数
          //在删除组件之前进行清理操作，比如计时器和事件监听器
          componentWillUnmount: function() {
              alert('销毁，我是组件');
          }
      }
    );

    ReactDOM.render(
      <HelloLifecycle />,
      document.getElementById('root')
    );
    </script>
</body>
```
运行上面的代码可以看到`console`打印出了了
1.  getDefaultProps,1
2.  getInitialState,2
3.  componentWillMount,3
4.  component render,4
5.  componentDidMount,5

这个是初始化时候的生命周期。运行时候的生命周期主要就是两部分 `update` `Unmount`,分别有组件更新前，组件更新后，组件销毁前，组件销毁后。其实总的来说分三种
1. Mounting  加载组件
2. Updating  重新记载组件
3. Unmounting  移除组件

针对这三种状态，`React`每种状态设置了`will`和`did`回调函数。分别代表前和后。具体的生命周期内容可以参考官方文档 `https://facebook.github.io/react/docs/react-component.html`

### 六 Refs and the DOM
```html
<body>
    <div id="root"></div>
    <script type="text/babel">
    class CustomTextInput extends React.Component {
      constructor(props) {
        super(props);
        //绑定focus回调函数
        this.focus = this.focus.bind(this);
      }

      focus() {
        // 获取输入框的焦点
        this.textInput.focus();
      }

      render() {
        return (
          <div>
            <input
              type="text"
              ref={(input) => { this.textInput = input; }} />
            <input
              type="button"
              value="Focus the text input"
              onClick={this.focus}
            />
          </div>
        );
      }
    }
    ReactDOM.render(
      <CustomTextInput />,
      document.getElementById('root')
    );
    </script>
</body>
```
在`React`中是基于数据流的，`props`是父组件和子组件交互的唯一方式，但是在某些情况下，我们需要在数据流之外强制修改子组件，子组件可以使`React`的实例，也可以是`DOM`元素。针对这种情况，可以使用`refs`属性。当组件装载的时候，`React`将会使用真实的`DOM`元素调用`ref`回调函数，当组件卸载的时候，会用`null`对象来回调，在上面的代码中，`CustomTextInput`组件装载后，`React`会调用`ref`的回调函数，将真实的`input`元素，复制给组件里面的`textInput`。当我们点击`button`的时候，调用`focus`回调，获取输入框的焦点。其实`React`也可以通过`this.refs.myRefName`来获取真实的`DOM`对象，官方建议我们使用事件回调的给属性赋值的方式，因为`this.refs.myRefName`必须在组件加载好了之后才能获取到真实的`DOM`，为了使用更加的友好，应该抛弃这种方式。再者</font color='red'>不可以在自定义的组件中使用ref属性，因为组件是没有具体的实例的</font>

### 总结
1.  `React`是基于状态驱使的。在整个`React`开发中，体会到，组件的状态贯穿着一切，一切都是基于状态驱使开发的。
2.  仅仅只要表达出应用程序在任一个时间点应该长的样子，然后当底层的数据变了，`React`会自动处理所有用户界面的更新。
3.  `React`中数据流是单向的，从父节点传递到子节点，因而组件是简单且易于把握的，组件只需要从父节点获取`props`渲染即可。如果顶层组件的某   个`prop`改变了，`React`会递归地向下遍历整棵组件树，重新渲染所有使用这个属性的组件。
4.  可以通过 `this.props` 来访问组件的 `props` ，但不能通过这种方式修改它，一个组件绝对不可以自己修改自己的 `props` 。 `props` 可以是字符串，对象，事件处理器等。可以使用 `PropTypes` 来验证 `props` 的类型，推荐使用。
5.  每一个`React`组件都可以拥有自己的`state`，其与`props`的区别是，`state`只存在于组件内部,`state` 的值应该通过 `this.setState()`修改，只要该状态被修改， `render` 方法就会被调用，然后导致之后的一系列变化。状态总是让组件变得更加复杂，把状态针对不同的组件独立开来，能更有利于调试。
