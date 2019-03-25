# 有关this
`this`是`Javascript`函数内部的一个特殊对象，引用的是函数运行时的环境对象，也就是说，`this`是动态的(箭头函数除外)，是在运行时进行绑定的，并不是在编写时绑定(箭头函数是编写时绑定)，它的上下文取决于函数调用时的各种条件。 `this`的绑定和函数声明的位置没有任何关系，只取决于函数的调用方式。 

函数有以下几种调用方式：
### 全局性调用
函数的最通常用法，`this`代表全局对象:
```javascript
function sayColor() {
  console.log(this.color)
}
var color = 'red'
sayColor()   // red

```