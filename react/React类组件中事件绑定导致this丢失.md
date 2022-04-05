# React类组件中事件绑定导致this丢失

### 问题

```react
import React, { Component } from 'react'

export default class Demo extends Component {

	constructor(props) {
		super(props)
		this.handleClick = this.handleClick.bind(this)
	}

	handleClick() {
		console.log(this) // Demo组件的实例对象
	}
	
	render() {
		return (
			<button onClick={this.handleClick}>Click me!</button>
		)
	}
}

```

```react
import React, { Component } from 'react'

export default class Demo extends Component {

	constructor(props) {
		super(props)
	}

	handleClick() {
		console.log(this) // undefined
	}
	
	render() {
		return (
			<button onClick={this.handleClick}>Click me!</button>
		)
	}
}

```



> 问题描述：在使用React类组件时，如果进行事件绑定，我们一般会用bind或者箭头函数对函数的this值进行绑定。那么为什么会出现this丢失的问题呢？

其实出现this丢失这种问题**不是React的问题，而是JavaScript的问题**。让我们简单回顾一下JavaScript中this的机制。

### JavaScript中this的指向问题

* 默认绑定（函数调用形式）

```js
function fun1() {
	console.log(this) // 严格模式：undefined  非严格模式：window
}
fun1()
```

这是一个普通的函数调用。this在非严格模式下指向window或者global对象。在严格模式下，this指向undefined。

* 隐式绑定（方法调用模式）

```js
let person = {
    name: 'Yuuki Asuna',
    printName: function() {
        console.log(this)
    }
}

person.printName() // {name: 'Yuuki Asuna', printName: ƒ}
```

当我们用person对象来调用printName函数时，this指向person对象。

**隐式绑定this丢失的情况**

```js
let person = {
    name: 'Yuuki Asuna',
    printName: function() {
        console.log(this)
    }
}
let print = person.printName
print() // window对象
```

当我们将函数的引用赋值给一个其他变量，当我们用这个新的变量去调用函数时，这是this又是window。

原因：**当我们调用print时，没有去指定一个具体的上下文对象。这是个没有所有者对象的纯函数调用。在这种情况下，printName内部的this值会退回到默认绑定。**

同理，如果我们将函数作为一个参数传到另一个函数内部调用，如下：

```js
let person = {
    name: 'Yuuki Asuna',
    printName: function() {
        console.log(this)
    }
}

function fun(callback) {
    callback()
}

fun(person.printName) // window对象
```

在传参的过程中，`callback = person.printName`会被执行，这样也导致printName失去宿主对象（即从方法调用变成了函数调用），即如果调用callback其中的this是指向window对象的（在严格模式指向undefined）。

* 明确绑定

那么如何解决隐式绑定（方法调用）变成默认绑定（函数调用）？

为了避免这种情况，我们可以使用**明确绑定方法**，将this的值通过bind方法绑定到函数上，这时候this的指向就不会发生改变

```js
let person = {
    name: 'Yuuki Asuna',
    printName: function() {
        console.log(this)
    }
}

let print = person.printName.bind(person)

print() // {name: 'Yuuki Asuna', printName: ƒ}
```

### React中this指向丢失的原因

说了这么多JavaScript中this指向问题，那和React中this指向丢失到底有什么关系呢？

```react
render() {
    return (
    	<button onClick={this.handleClick}>Click me!</button>
    )
}
```

其实是**jsx**导致this指向丢失的，实际上，JSX仅仅是`React.createElement(component, props, ...children)`函数的语法糖，所以上面的代码和下面的代码可以看成相同的。

```react
React.createElement("button", {
  onClick: this.handleClick
}, "Click me!");
```

我们可以很清楚的看到`React.createElement(component, props, ...children)`在传入第二个参数时进行了赋值操作，导致了this变成默认绑定，在严格模式下，this就指向了undefined。这就是React类组件中绑定事件函数会导致this指向丢失的原因。

### 解决方法

* 利用`this.handleClick = this.handleClick.bind(this)`将该函数与当前实例对象绑定，改变this指向
* 箭头函数：箭头函数不会绑定this，它会捕获其所在（即定义的位置）上下文的this，作为自己的this值
  * 在声明时，将事件函数声明为箭头函数
  * 在传递事件函数的时候，传递一个箭头函数`<button onClick={() => { this.handleClick() }>Click me!</button>`

### 参考

> 1. [[译] 为什么需要在 React 类组件中为事件处理程序绑定 this](https://juejin.cn/post/6844903605984559118)
> 2. [[React] 绑定事件为什么会this丢失](https://blog.csdn.net/CSDNzhaojiale/article/details/109690670)