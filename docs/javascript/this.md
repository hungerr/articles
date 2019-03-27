# 有关this
`this`是`Javascript`函数内部的一个特殊对象，引用的是函数运行时的环境对象，也就是说，`this`是动态的(箭头函数除外)，是在运行时进行绑定的，并不是在编写时绑定(箭头函数是编写时绑定)。 `this`的绑定和函数声明的位置没有任何关系，只取决于函数的调用方式。 
## 绑定规则
`this`绑定根据函数的调用方式基本上有四种规则：
### 全局性调用
函数的最通常用法，`this`代表全局对象：
```javascript
function sayColor() {
  console.log(this.color)
}
var color = 'red'
sayColor()   // red
```
### 作为对象的方法调用
函数作为某个对象的方法调用时，`this`指向这个上级对象：
```javascript
let car = {
  color: 'black',
  sayColor
}
car.sayColor()  // black
```
当存在多重调用时，最后一层会决定调用位置：
```javascript
let house = {
  color: 'white',
  car
}
house.car.sayColor()  // black
```
### 使用apply、call、bind方法绑定调用对象
函数可以使用`apply、call、bind`方法来改变调用对象。这些方法的第一个参数是一个对象，它们会把这个对象绑定到`this`：
```javascript
let bindObj = {
  color: 'green'
}
sayColor.apply(bindObj)  // green
sayColor.call(bindObj)  // green
sayColor.bind(bindObj)()  // green
```
第一个参数为空时，默认绑定全局对象：
```javascript
sayColor.apply()  // red
```
### 构造函数调用
当函数作为构造函数被调用时，`this`指向创建的新对象：
```javascript
function Person(color) {
  this.color = color
}
let p = new Person('yellow')
p.color  // yellow
```
使用new操作符调用构造函数时，首先会创建一个新对象，然后将构造函数的作用域赋给新对象，`this`就指向了新对象，过程与下面代码类似：
```javascript
let o = new Object()
Person.call(o, 'yellow')
o.color  // yellow
```
当然，构造函数不通过new调用时，与普通函数无区别，可以理解为实际上并不存在所谓的**构造函数**，只有对于函数的**构造调用**：
```javascript
Person('yellow')
// this指向了全局对象window
window.color  // yellow
```
以上就是`this`绑定的四种规则，函数调用时，先找到调用位置，然后判断需要应用四条规则中的哪一条。
## 应用
现在来看下实际应用的几种特殊情况。
### 回调函数
当做回调函数调用时，一般是全局调用：
```javascript
setTimeout(sayColor) // red

let obj = {
    color: 'green',
    say () {
        sayColor()
    }
}
obj.say() // red
```
但在一些上下文中会进行隐式绑定,比如事件中的`this`是指向于事件的目标元素的,还有一些数组的操作方法可以使用第二个参数来绑定`this`：
```javascript
[1, 2, 3].forEach(function () { console.log(this.color) })  // red red red
[1, 2, 3].forEach(function () { console.log(this.color) }, car)  // black black black
```

### 绑定丢失
```javascript
function sayColor() {
  console.log(this.color)
}
let car = {
  color: 'black',
  sayColor
}
var color = 'red'

let alias = car.sayColor
alias() // red
```
上述代码中将对象的方法赋值给新变量`alias`，`alias`函数执行时，`this`指向了全局对象。
函数的名字仅仅是一个包含指向函数对象的指针的变量，car对象的sayColor属性保存在一个属性描述符对象中：
```javascript
Object.getOwnPropertyDescriptor(car, 'sayColor')
//  {value: ƒ, writable: true, enumerable: true, configurable: true}
```
其中描述符对象的value属性保存了指向sayColor函数的指针，`let alias = car.sayColor`语句将指向sayColor函数的指针赋值给变量`alias`，执行`alias`函数就是在全局对象中执行函数`sayColor`。
当做回调函数时：
```javascript
setTimeout(car.sayColor, 500)  // red
```
因为Javascript中的函数参数都是按值传递的，上述代码将指向`sayColor`函数的指针赋值给了`setTimeout`函数的参数，也相当于在全局环境中执行`sayColor`函数。

### 闭包
匿名函数的执行一般具有全局性，在闭包中由于编写方式可能不会那么明显：
```javascript
let obj = {
  color: 'green',
  say () {
    return function() {
      console.log(this.color)
    }
  }
}
obj.say()()  // red   this指向全局window
```
内部匿名函数是有自己的`this`变量的，所以无法访问到外部函数`say`的`this`变量，我们可以将外部`this`变量保存于一个闭包能够访问的变量之中：
```javascript
let obj = {
  color: 'green',
  say () { 
    let self = this
    return function() {
      console.log(self.color)
    }
  }
}
obj.say()()  // green
```
当然，现在可以用箭头函数来绑定this：
```javascript
let obj = {
  color: 'green',
  say () { 
    return () => {
      console.log(this.color)
    }
  }
}
obj.say()()  // green
```
## 优先级
全局性调用优先级是最低的。
使用`apply`等函数绑定`this`的优先级高于对象调用：
```javascript
let car = {
  color: 'black',
  sayColor
}
let bindObj = {
  color: 'green'
}

car.sayColor()  // black
car.sayColor.apply(bindObj)  // green
```
使用`new`操作符绑定高于使用`apply`等函数：
```javascript
function Person(color) {
  this.color = color
}
let obj = {}
let bindPerson = Person.bind(obj)
let p = new bindPerson('yellow')

p.color  // yellow
obj.color  // undefined
```
## 箭头函数与this
ES6中引进了箭头函数，可以简化匿名函数的语法：
```javascript
setTimeout(() => { console.log(this.color) }, 50)
```
箭头函数内部是没有`this`、`arguments`、`super`、`new.target`特殊变量的，访问它们时会指向最近的外层非箭头函数的相应变量：
```javascript
function sayColor() {
  return () => {
    return () => {
      console.log(this.color)
    }
  }
}

sayColor.call({ color: 'red' })()()  // red 指向了外层sayColor函数的this对象
sayColor.call({ color: 'red' }).call({ color: 'green' })()  // red 依然指向外层sayColor函数的this对象
sayColor.call({ color: 'red' }).call({ color: 'green' }).call({ color: 'yellow' })  // red箭头函数使用call是无法绑定this的
```
所以，箭头函数可以起到固定化`this`指向的效果，一定程度上可以说`this`是静态的，参考上面闭包的代码：
```javascrip
// ES6箭头函数
let obj = {
  color: 'green',
  sayColor () { 
    return () => {
      console.log(this.color)
    }
  }
}

// ES5
let obj = {
  color: 'green',
  sayColor () { 
    let self = this
    return function() {
      console.log(self.color)
    }
  }
}
```
当然，静态并不意味着箭头函数的`this`是永远不变的，而是随着外层函数的`this`变化而变化：
```javascrip
let obj = {
  color: 'green',
  sayColor () { 
    return () => {
      console.log(this.color)
    }
  }
}

obj.sayColor()()  // green
obj.sayColor.call({ color: 'red' })()  // red
```
### 不适用情况
在事件中想将`this`指向目标元素时，箭头函数是不适用的：
```javascrip
btn.addEventListener('click', () => {
  console.log(this)
})
```
上述代码中`this`指向了全局对象，而不是事件的目标元素。

将函数当做对象的方法调用并且想将`this`指向对象时，也是不适用的：
```javascrip
let obj = {
  color: 'green',
  sayColor: () => { 
    console.log(this.color)
  }
}
```
上述代码中`this`也指向了全局对象。

总之，需要`this`动态时使用非箭头函数，需要`this`静态时使用箭头函数：
```javascrip
function Person() {
  this.color = 'yellow'
  setTimeout(() => { console.log('person color is',this.color) }, 50)
  setTimeout(function() { console.log('global color is',this.color) }, 50)
}
var color = 'red'
new Person()
//  输出
person color is yellow
global color is red
```