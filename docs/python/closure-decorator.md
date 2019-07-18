## Python装饰器与闭包

闭包是Python装饰器的基础。要理解闭包，先要了解Python中的变量作用域规则。

### 变量作用域规则

首先，在函数中是能访问全局变量的：

```python
>>> a = 'global var'

>>> def foo():
	print(a)

>>> foo()
global var
```

然后，在一个嵌套函数中：

```python
>>> def foo():
	a = 'free var'
	def bar():
	    print(a)
	return bar

>>> foo()()
free var
```

在函数bar中能够访问外层函数foo中定义的a变量。

### 闭包

上述情况就是闭包。**闭包**是指延伸了作用域的函数，其中能够访问未在函数定义体中定义的非全局变量。未在函数定义体中定义的非全局变量一般都是在嵌套函数中出现的。

上述示例中的变量a就是一个并未在函数bar中定义的非全局变量。对于bar来说，它有个专业名字，叫做自由变量。

自由变量的名称可以在字节码对象中查看：

```python
>>> bar = foo()
>>> bar.__code__.co_freevars
('a',)
```

值绑定在函数的__closure__属性中：

```python
>>> bar.__closure__
(<cell at 0x000001CB2912DF48: str object at 0x000001CB291D3D70>,)
```

其中保存了对应自由变量的cell对象的序列，cell对象的cell_contents属性保存了变量的值：

```python
>>> bar.__closure__[0].cell_contents
'free var'
```

这与JavaScript中闭包的行为是类似的，JavaScript中函数会将外层函数的活动对象添加到它的作用域链中。但与JavaScript不同的是，Python在函数中是无法直接改变全局变量与自由变量的值的：

```python
>>> a = 1
>>> def foo():
	print(a)
	a += 1

>>> foo()
UnboundLocalError: local variable 'a' referenced before assignment

>>> def foo():
	a = 1
	def bar():
	    print(a)
	    a += 1
	return bar

>>> foo()()
UnboundLocalError: local variable 'a' referenced before assignment
```

两种情况下，都会报错。这并不是缺陷，而是Python的设计选择。Python不要求声明变量，但是会假定在函数定义体重赋值的变量是局部变量，以避免在不知情情况下修改全局变量。

`a += 1`与`a = a + 1`相同，编译函数的定义体时，会将a当做局部变量，但尝试获取a的值时，发现a并没有绑定值，于是报错。

解决这个问题的办法，一是将变量置于一些可变对象，如列表、字典中，另外的方法就是使用**global**或者**nonlocal**将变量声明为全局变量或者自由变量：

```python
>>> def foo():
	a = 1
	def bar():
	    nonlocal a
	    a += 1
	    print(a)
	return bar

>>> foo()()
2
```

### 装饰器