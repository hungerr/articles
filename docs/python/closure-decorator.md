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

然后，在一个嵌套函数中，内层函数能够访问在外层函数中定义的局部变量：

```python
>>> def foo():
        a = 'free var'
        def bar():
            print(a)
        return bar

>>> foo()()
free var
```

### 闭包

上面的嵌套函数就是闭包。**闭包**是指延伸了作用域的函数，在其中能够访问未在函数定义体中定义的非全局变量。未在函数定义体中定义的非全局变量一般都是在嵌套函数中出现的。

上述示例中的变量a就是一个并未在函数bar中定义的非全局变量。对于bar来说，它有个专业名字，叫做**自由变量**。

自由变量的名称可以在字节码对象中查看：

```python
>>> bar = foo()
>>> bar.__code__.co_freevars
('a',)
```

自由变量的值绑定在函数的__closure__属性中：

```python
>>> bar.__closure__
(<cell at 0x000001CB2912DF48: str object at 0x000001CB291D3D70>,)
```

其中保存了对应自由变量的cell对象的序列，cell对象的cell_contents属性保存了变量的值：

```python
>>> bar.__closure__[0].cell_contents
'free var'
```

这与JavaScript中闭包的行为是类似的，JavaScript中嵌套函数会将外层函数的活动对象添加到它的作用域链中。但与JavaScript不同的是，当Python函数中的全局变量或者自由变量是不可变对象(数字、字符串、元组等)时，是只能读取，无法更新的：

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

两种情况下，都会报错。这并不是缺陷，而是Python的设计选择。Python不要求声明变量，但是会假定在函数定义体中赋值的变量是局部变量，以避免在不知情的情况下修改全局变量。

`a += 1`与`a = a + 1`相同，编译函数的定义体时，会将a当做局部变量，不会当做自由变量保存。然后尝试获取a的值时，发现a并没有绑定值，于是报错。

解决这个问题的办法，一是将变量置于一些可变对象，如列表、字典中：

```python
def foo():
    ns = {}
    ns['a'] = 1
    def bar():
        ns['a'] += 1
        print (ns['a'])
    return bar
```

另外的方法就是使用**global**或者**nonlocal**将变量声明为全局变量或者自由变量：

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

当自由变量本身是可变对象时，是可以直接进行操作的：

```python
def make_avg():
    ls = []
    def avg(x):
        ls.append(x)
        print(sum(ls)/len(ls))
    return avg
```

### 延迟绑定
需要注意的一点是，python函数的作用域是由代码决定的，也就是静态的，但它们的使用是动态的，是在执行时确定的。
```python
>>> fs = [lambda x: x * i for i in range(4)]
>>> for foo in fs:
	    print(foo(1))

	
3
3
3
3
```
当你期待结果是0的时候，结果却是3。

这是因为只有在函数foo被执行的时候才会搜索变量i的值, 由于循环已结束, i指向最终值3, 所以都会得到相同的结果。

在闭包中也存在相同的问题：

```python
def foo():
    fs = []
    for i in range(4):
        fs.append(lambda x: x*i)
    return fs
for f in foo():
    print(f(1))
```
返回：
```python
3
3
3
3
```

解决方法，一个是为函数参数设置默认值：
```python
>>> fs = [lambda x, i=i: x * i for i in range(4)]
>>> for f in fs:
        print(f(1))
```

另外就是使用闭包了：
```python
>>> def foo(i):
        return lambda x: x * i
 
>>> fs = [foo(i) for i in range(4)]
>>> for f in fs:
        print(f(1))
```

或者：
```PYTHON
>>> for f in map(lambda i: lambda x: i*x, range(4)):
        print(f(1))
```

使用闭包就很类似于偏函数了，也可以使用偏函数:

```PYTHON
>>> fs = [functools.partial(lambda x, i: x * i, i) for i in range(4)]
>>> for f in fs:
        print(f(1))
```

这样自由变量i都会优先绑定到闭包函数上。

### 装饰器

装饰器是可调用对象，参数一般是另一个函数。装饰器可以以某种方式增强被装饰函数的行为，然后返回被装饰的函数或者将其替换成一个新的函数。

一个最简单的不做任何额外行为的装饰器：

```python
def decorate(func):
    return func
```

`decorate`函数就是一个最简单的装饰器，使用方法：

```python
def target():
    pass

target = decorate(target)
```

Python为装饰器的使用提供了语法糖，可以简便的写为：

```python
@decorate
def target():
    pass
```

#### 导入时运行

装饰器一个很重要的特性是它是导入时(加载模块时)运行的：

```python
def decorate(func):
    print('running decorator when import')
    return func

@decorate
def foo():
    print('running foo')
    pass

if __name__ == '__main__':
    print('start foo')
    foo()
```

结果：

```python
running decorator when import
start foo
running foo
```

可以看到，装饰器是导入时运行的，而被装饰的函数是明确调用时运行的。

装饰器可以返回被装饰的函数本身，和运行时导入的特性结合起来，可以实现简单的注册器功能：

```python
view_registry = []

def register(func):
    view_registry.append(func)
    return func

@register
def view1():
    pass

@register
def view2():
    pass

def main():
    print(view_registry)


if __name__ == '__main__':
    main()
```

#### 返回新函数

上述装饰器的例子都返回了被装饰的原函数，但装饰器的典型行为还是返回一个新函数：把被装饰的函数替换成新函数，新函数接受与原函数相同的参数，并且返回原函数本该返回的值。写法类似于：

```python
def deco(func):
    def new_func(*args, **kwargs):
        return func(*args, **kwargs)
    return new_func
```

这种情况下装饰器就使用到了闭包。JavaScript中的防抖与节流函数就是这种典型的装饰器行为。新函数一般会使用外部装饰器函数中的变量当做自由变量，对函数作出某种增强行为。

举个例子，我们知道，当Python函数的参数是个可变对象时，会产生意料之外的行为：

```python
def foo(x, y=[]):
    y.append(x)
    print(y)

foo(1)
foo(2)
foo(3)
```

输出：

```python
[1]
[1, 2]
[1, 2, 3]
```

这是因为，函数的参数默认值保存在__defaults__属性中，指向了同一个列表：

```python
>>> foo.__defaults__
([1, 2, 3],)
```

我们就可以用一个装饰器在函数执行前取出默认值做深复制，然后覆盖函数原先的参数默认值：

```python
import copy

def fresh_defaults(func):
    defaults = func.__defaults__
    def deco(*args, **kwargs):
        func.__defaults__ = copy.deepcopy(defaults)
        return func(*args, **kwargs)
    return deco

@fresh_defaults
def foo(x, y=[]):
    y.append(x)
    print(y)

foo(1)
foo(2)
foo(3)
```
输出：
```
[1]
[2]
[3]
```

#### 接收参数的装饰器

装饰器除了可以接受函数作为参数外，还可以接受其他参数。使用方法是：创建一个装饰器工厂，接受参数，返回一个装饰器，再把它应用到被装饰的函数上，语法如下：

```python
def deco_factory(*args, **kwargs):
    def deco(func):
        print(args)
        return func
    return deco

@deco_factory('factory')
def foo():
    pass
```

在Web框架中，通常要将URL模式映射到生成响应的view函数，并将view函数注册到某些中央注册处。之前我们曾经实现过一个简单的注册装饰器，只是注册了view函数，却没有URL映射，是远远不够的。

在Flask中，注册view函数需要一个装饰器：

```python
@app.route('/hello')
def hello():
    return 'Hello, World'
```

原理就是使用了装饰器工厂，可以简单的模拟一下实现：

```python
class App:
    def __init__(self):
        self.view_functions = {}

    def route(self, rule):
        def deco(view_func):
            self.view_functions[rule] = view_func
            return view_func
        return deco
         
app = App()

@app.route('/')
def index():
    pass

@app.route('/hello')
def hello():
    pass

for rule, view in app.view_functions.items():
    print(rule, ':', view.__name__)
```
输出：
```python
/ : index
/hello : hello
```

还可以使用装饰器工厂来确定view函数可以允许哪些HTTP请求方法：

```python
def action(methods):
    def deco(view):
        view.allow_methods = [method.lower() for method in methods]
        return view
    return deco
         
@action(['GET', 'POST'])
def view(request):
    if request.method.lower() in view.allow_methods:
        ...
```

#### 重叠的装饰器

装饰器也是可以重叠使用的：

```python
@d1
@d2
def foo():
    pass
```
等同于：
```python
foo = d1(d2(foo))
```

#### 类装饰器

装饰器的参数也可以是一个类，也就是说，装饰器可以装饰类：

```python
import types

def deco(cls):
    for key, method in cls.__dict__.items():
        if isinstance(method, types.FunctionType):
            print(key, ':', method.__name__)
    return cls

@deco
class Test:
    def __init__(self):
        pass

    def foo(self):
        pass
```
