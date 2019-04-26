# Python Web开发中的WSGI协议简介
在Python Web开发中，我们一般使用`Flask`、`Django`等web框架来开发应用程序，生产环境中将应用部署到`Apache`、`Nginx`等web服务器时，还需要`uWSGI`或者`Gunicorn`。一个完整的部署应该类似这样：

```
Web Server(Nginx、Apache) <-----> WSGI server(uWSGI、Gunicorn) <-----> App(Flask、Django)
```

要弄清这些概念之间的关系，就需要先理解`WSGI`协议。
## WSGI是什么
`WSGI`的全称是`Python Web Server Gateway Interface`，`WSGI`不是web服务器，python模块，或者web框架以及其它任何软件，它只是一种规范，描述了`web server`如何与`web application`进行通信的规范。[PEP-3333](https://www.python.org/dev/peps/pep-3333/ "PEP-3333")有关于`WSGI`的具体定义。
## 为什么需要WSGI
我们使用web框架进行web应用程序开发时，只专注于业务的实现，HTTP协议层面相关的事情交于web服务器来处理，那么，Web服务器和应用程序之间就要知道如何进行交互。有很多不同的规范来定义这些交互，最早的一个是CGI，后来出现了改进CGI性能的FasgCGI。Java有专用的`Servlet`规范，实现了Servlet API的Java web框架开发的应用可以在任何实现了Servlet API的web服务器上运行。`WSGI`的实现受`Servlet`的启发比较大。
## WSGI的实现
在`WSGI`中有两种角色：一方称之为`server`或者`gateway`, 另一方称之为`application`或者`framework`。`application`可以提供一个可调用对象供`server`调用。`server`先收到用户的请求，然后调用`application`提供的可调用对象，调用的结果会被封装成HTTP响应后发送给客户端。
### The Application/Framework Side
`WSGI`对`application`的要求有3个：

- 实现一个可调用对象
- 可调用对象接收两个参数，`environ`（一个`dict`，包含`WSGI`的环境信息）与`start_response`（一个响应请求的函数）
- 返回一个`iterable`可迭代对象

可调用对象可以是一个函数、类或者实现了`__call__`方法的类实例。

`environ`和`start_response`由`server`方提供。

`environ`是包含了环境信息的字典。

`start_response`也是一个callable，接受两个必须的参数，`status`（HTTP状态）和`response_headers`(响应消息的头)，可调用对象返回前调用`start_response`。

下面是[PEP-3333](https://www.python.org/dev/peps/pep-3333//#the-application-framework-side "PEP-3333")实现简单`application`的代码：
```python
HELLO_WORLD = b"Hello world!\n"

def simple_app(environ, start_response):
    """Simplest possible application object"""
    status = '200 OK'
    response_headers = [('Content-type', 'text/plain')]
    start_response(status, response_headers)
    return [HELLO_WORLD]

class AppClass:

    def __init__(self, environ, start_response):
        self.environ = environ
        self.start = start_response

    def __iter__(self):
        status = '200 OK'
        response_headers = [('Content-type', 'text/plain')]
        self.start(status, response_headers)
        yield HELLO_WORLD
```
