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
代码分别用函数与类对`application`的可调用对象进行了实现。类实现中定义了`__iter__`方法，返回的类实例就变为了iterable可迭代对象。

我们也可以用定义了__call__方法的类实例做一下实现：
```python
class AppClass:

    def __call__(self, environ, start_response):
        status = '200 OK'
        response_headers = [('Content-type', 'text/plain')]
        start_response(status, response_headers)
        return [HELLO_WORLD]
```

### The Server/Gateway Side
`server`要做的是每次收到HTTP请求时，调用`application`可调用对象，pep文档代码:
```python
import os, sys

enc, esc = sys.getfilesystemencoding(), 'surrogateescape'

def unicode_to_wsgi(u):
    # Convert an environment variable to a WSGI "bytes-as-unicode" string
    return u.encode(enc, esc).decode('iso-8859-1')

def wsgi_to_bytes(s):
    return s.encode('iso-8859-1')

def run_with_cgi(application):
    environ = {k: unicode_to_wsgi(v) for k,v in os.environ.items()}
    environ['wsgi.input']        = sys.stdin
    environ['wsgi.errors']       = sys.stderr
    environ['wsgi.version']      = (1, 0)
    environ['wsgi.multithread']  = False
    environ['wsgi.multiprocess'] = True
    environ['wsgi.run_once']     = True

    if environ.get('HTTPS', 'off') in ('on', '1'):
        environ['wsgi.url_scheme'] = 'https'
    else:
        environ['wsgi.url_scheme'] = 'http'

    headers_set = []
    headers_sent = []

    def write(data):
        out = sys.stdout

        if not headers_set:
             raise AssertionError("write() before start_response()")

        elif not headers_sent:
             # Before the first output, send the stored headers
             status, response_headers = headers_sent[:] = headers_set
             out.write(wsgi_to_bytes('Status: %s\r\n' % status))
             for header in response_headers:
                 out.write(wsgi_to_bytes('%s: %s\r\n' % header))
             out.write(wsgi_to_bytes('\r\n'))

        out.write(data)
        out.flush()

    def start_response(status, response_headers, exc_info=None):
        if exc_info:
            try:
                if headers_sent:
                    # Re-raise original exception if headers sent
                    raise exc_info[1].with_traceback(exc_info[2])
            finally:
                exc_info = None     # avoid dangling circular ref
        elif headers_set:
            raise AssertionError("Headers already set!")

        headers_set[:] = [status, response_headers]

        # Note: error checking on the headers should happen here,
        # *after* the headers are set.  That way, if an error
        # occurs, start_response can only be re-called with
        # exc_info set.

        return write

    result = application(environ, start_response)
    try:
        for data in result:
            if data:    # don't send headers until body appears
                write(data)
        if not headers_sent:
            write('')   # send headers now if body was empty
    finally:
        if hasattr(result, 'close'):
            result.close()
```

代码中`server`组装了`environ`，定义了`start_response`函数，将两个参数提供给`application`并且调用，最后输出HTTP响应的`status`、`header`和`body`:
```python
if __name__ == '__main__':
    run_with_cgi(simple_app)
```
输出：
```
Status: 200 OK
Content-type: text/plain

Hello world!
```

### environ
environ字典包含了一些CGI规范要求的数据，以及WSGI规范新增的数据，还可能包含一些操作系统的环境变量以及Web服务器相关的环境变量，具体见[environ](https://www.python.org/dev/peps/pep-3333/#environ-variables)。

我们可以使用python官方库wsgiref实现的`server`看一下`environ`的具体内容：
```python
def demo_app(environ, start_response):
    from StringIO import StringIO
    stdout = StringIO()
    print >>stdout, "Hello world!"
    print >>stdout
    h = environ.items()
    h.sort()
    for k,v in h:
        print >>stdout, k,'=', repr(v)
    start_response("200 OK", [('Content-Type','text/plain')])
    return [stdout.getvalue()]

if __name__ == '__main__':
    from wsgiref.simple_server import make_server
    httpd = make_server('', 8000, demo_app)
    sa = httpd.socket.getsockname()
    print "Serving HTTP on", sa[0], "port", sa[1], "..."
    import webbrowser
    webbrowser.open('http://localhost:8000/xyz?abc')
    httpd.handle_request()  # serve one request, then exit
    httpd.server_close()
```
打开的网页会输出`environ`的具体内容。

使用`environ`组装请求URL地址：
```python
from urllib.parse import quote
url = environ['wsgi.url_scheme']+'://'

if environ.get('HTTP_HOST'):
    url += environ['HTTP_HOST']
else:
    url += environ['SERVER_NAME']

    if environ['wsgi.url_scheme'] == 'https':
        if environ['SERVER_PORT'] != '443':
           url += ':' + environ['SERVER_PORT']
    else:
        if environ['SERVER_PORT'] != '80':
           url += ':' + environ['SERVER_PORT']

url += quote(environ.get('SCRIPT_NAME', ''))
url += quote(environ.get('PATH_INFO', ''))
if environ.get('QUERY_STRING'):
    url += '?' + environ['QUERY_STRING']
```

### start_resposne
`start_response`是一个可调用对象，接收两个必选参数和一个可选参数:

- `status`: 一个字符串，表示HTTP响应状态字符串，比如`200 OK`、`404 Not Found`
- `headers`: 一个列表，包含有如下形式的元组：`(header_name, header_value)`，用来表示HTTP响应的`headers`
- `exc_info`（可选）: 用于出错时，`server`需要返回给浏览器的信息

`start_response`必须返回一个[`write(body_data)`](https://www.python.org/dev/peps/pep-3333/#the-write-callable)可调用对象。

我们知道HTTP的响应需要包含`status`，`headers`和`body`，所以在`application`对象将`body`作为返回值return之前，需要先调用`start_response`，将`status`和`headers`的内容返回给`server`，这同时也是告诉`server`，`application`对象要开始返回body了。

### 关系
由此可见，`server`负责接收HTTP请求，根据请求数据组装`environ`，定义`start_response`函数，将这两个参数提供给`application`。`application`根据`environ`信息执行业务逻辑，将结果返回给`server`。响应中的`status`、`headers`由`start_response`函数返回给`server`，响应的`body`部分被包装成`iterable`作为`application`的返回值`，`server`将这些信息组装为HTTP响应返回给请求方。

在一个完整的部署中，`uWSGI`和`Gunicorn`是实现了`WSGI`的`server`，`Django`、`Flask`是实现了`WSGI`的`application`。两者结合起来其实就能实现访问功能。实际部署中还需要`Nginx`、`Apache`的原因是它有很多`uWSGI`没有支持的更好功能，比如处理静态资源，负载均衡等。`Nginx`、`Apache`一般都不会内置`WSGI`的支持，而是通过扩展来完成。比如`Apache`服务器，会通过扩展模块`mod_wsgi`来支持`WSGI`。`Apache`和`mod_wsgi`之间通过程序内部接口传递信息，`mod_wsgi`会实现`WSGI`的`server`端、进程管理以及对`application`的调用。`Nginx`上一般是用proxy的方式，用`Nginx`的协议将请求封装好，发送给应用服务器，比如`uWSGI`，`uWSGI`会实现`WSGI`的服务端、进程管理以及对`application`的调用。

### Middleware
`WSGI`除了`server`和`application`两个角色外，还有`middleware`中间件，`middleware`运行在`server`和`application`中间,同时具备`server`和`application`的角色，对于`server`来说，它是一个`application`；对于`application`来说，它是一个`server`，文档：
```python
from piglatin import piglatin

class LatinIter:

    """Transform iterated output to piglatin, if it's okay to do so

    Note that the "okayness" can change until the application yields
    its first non-empty bytestring, so 'transform_ok' has to be a mutable
    truth value.
    """

    def __init__(self, result, transform_ok):
        if hasattr(result, 'close'):
            self.close = result.close
        self._next = iter(result).__next__
        self.transform_ok = transform_ok

    def __iter__(self):
        return self

    def __next__(self):
        if self.transform_ok:
            return piglatin(self._next())   # call must be byte-safe on Py3
        else:
            return self._next()

class Latinator:

    # by default, don't transform output
    transform = False

    def __init__(self, application):
        self.application = application

    def __call__(self, environ, start_response):

        transform_ok = []

        def start_latin(status, response_headers, exc_info=None):

            # Reset ok flag, in case this is a repeat call
            del transform_ok[:]

            for name, value in response_headers:
                if name.lower() == 'content-type' and value == 'text/plain':
                    transform_ok.append(True)
                    # Strip content-length if present, else it'll be wrong
                    response_headers = [(name, value)
                        for name, value in response_headers
                            if name.lower() != 'content-length'
                    ]
                    break

            write = start_response(status, response_headers, exc_info)

            if transform_ok:
                def write_latin(data):
                    write(piglatin(data))   # call must be byte-safe on Py3
                return write_latin
            else:
                return write

        return LatinIter(self.application(environ, start_latin), transform_ok)


# Run foo_app under a Latinator's control, using the example CGI gateway
from foo_app import foo_app
run_with_cgi(Latinator(foo_app))
```
可以看出，`Latinator`调用`foo_app`充当`server`角色，然后实例被`run_with_cgi`调用充当`application`角色。

## Django中WSGI的实现
每个Django项目中都有个wsgi.py文件，作为`application`是这样实现的：
```python
from django.core.wsgi import get_wsgi_application
application = get_wsgi_application()
```
源码：
```python
from django.core.handlers.wsgi import WSGIHandler


def get_wsgi_application():
    """
    The public interface to Django's WSGI support. Return a WSGI callable.
    Avoids making django.core.handlers.WSGIHandler a public API, in case the
    internal WSGI implementation changes or moves in the future.
    """
    django.setup(set_prefix=False)
    return WSGIHandler()
```
`WSGIHandler`:
```python
class WSGIHandler(base.BaseHandler):
    request_class = WSGIRequest

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.load_middleware()

    def __call__(self, environ, start_response):
        set_script_prefix(get_script_name(environ))
        signals.request_started.send(sender=self.__class__, environ=environ)
        request = self.request_class(environ)
        response = self.get_response(request)

        response._handler_class = self.__class__

        status = '%d %s' % (response.status_code, response.reason_phrase)
        response_headers = [
            *response.items(),
            *(('Set-Cookie', c.output(header='')) for c in response.cookies.values()),
        ]
        start_response(status, response_headers)
        if getattr(response, 'file_to_stream', None) is not None and environ.get('wsgi.file_wrapper'):
            response = environ['wsgi.file_wrapper'](response.file_to_stream)
        return response
```
`application`是一个定义了__call__方法的`WSGIHandler`类实例，首先加载中间件，然后根据`environ`生成请求`request`，根据请求生成响应`response`，`status`和`response_headers`由`start_response`处理，然后返回响应body。

运行`runserver`命令时，django可以自身起一个`WSGI server`,[`django/core/servers/basehttp.py`](https://github.com/django/django/blob/master/django/core/servers/basehttp.py)文件：
```python
def run(addr, port, wsgi_handler, ipv6=False, threading=False, server_cls=WSGIServer):
    server_address = (addr, port)
    if threading:
        httpd_cls = type('WSGIServer', (socketserver.ThreadingMixIn, server_cls), {})
    else:
        httpd_cls = server_cls
    httpd = httpd_cls(server_address, WSGIRequestHandler, ipv6=ipv6)
    if threading:
        # ThreadingMixIn.daemon_threads indicates how threads will behave on an
        # abrupt shutdown; like quitting the server by the user or restarting
        # by the auto-reloader. True means the server will not wait for thread
        # termination before it quits. This will make auto-reloader faster
        # and will prevent the need to kill the server manually if a thread
        # isn't terminating correctly.
        httpd.daemon_threads = True
    httpd.set_app(wsgi_handler)
    httpd.serve_forever()
```
实现的`WSGIServer`，继承自`wsgiref`:
```python
class WSGIServer(simple_server.WSGIServer):
    """BaseHTTPServer that implements the Python WSGI protocol"""

    request_queue_size = 10

    def __init__(self, *args, ipv6=False, allow_reuse_address=True, **kwargs):
        if ipv6:
            self.address_family = socket.AF_INET6
        self.allow_reuse_address = allow_reuse_address
        super().__init__(*args, **kwargs)

    def handle_error(self, request, client_address):
        if is_broken_pipe_error():
            logger.info("- Broken pipe from %s\n", client_address)
        else:
            super().handle_error(request, client_address)
```