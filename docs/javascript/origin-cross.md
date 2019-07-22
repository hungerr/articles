## 同源策略及跨域

同源策略（same-origin policy）是浏览器安全的基石。Web是构建在同源策略基础之上的，浏览器只是针对同源策略的一种实现。

1995年，同源策略由 Netscape 公司引入浏览器。目前，所有浏览器都实行这个策略。

### 同源
所谓同源，是指三个相同：

- 协议相同
- 域名相同
- 端口相同

比如，`https://www.github.com/hungerr`，协议是`https://`，域名是`www.github.com`，端口是`80`。

同源政策的目的，是为了保证用户信息的安全，防止恶意的网站窃取数据。

网站一般是以Cookie来识别用户的，如果可以获取非同源的Cookie，就可以冒充网站用户，为所欲为了。用户获取所有的Cookie，然后发往一个网站，此网站就可能获取此用户在其它网站的信息或者权限等。

### 限制

三个有一个不一样则为非同源，共有三种行为受到限制：

- Cookie、LocalStorage 和 IndexDB 无法读取。
- DOM 无法获得。
- AJAX 请求不能发送。

### Cookie

一般情况下，二级域名不同的网址是不能共享Cookie，但可以通过设置`document.domain`来共享Cookie。

举例来说，A网页是`http://a.example.com/`，B网页是`http://b.example.com/`，那么只要设置相同的`document.domain`，两个网页就可以共享Cookie，设置：
```javascript
document.domain = 'example.com';
```
现在，A网页通过脚本设置一个 Cookie:
```javascript
document.cookie = "name=value";
```
B网页就可以读到这个 Cookie:
```javascript
var shareCookie = document.cookie;
```
另外，服务器也可以在设置Cookie的时候，指定Cookie的domain：
```javascript
Set-Cookie: name=value; domain=.example.com
```
设置为`.example.com.`时，对所有的子域都有效。 如果没有明确设定，那么这个域会被认作来自设置 cookie 的那个域。

### AJAX与跨域

同源政策规定，AJAX请求只能发给同源的网址。前后端分离的情况下，实现合理的跨域请求对开发某些浏览器应用程序也是至关重要的。

#### CORS
CORS（Cross-Origin Resource Sharing，跨源资源共享）是 W3C 的 一个工作草案，定义了在必须访问跨源资源时，浏览器与服务器应该如何沟通。 CORS 背后的基本思想，就是使用自定义的 HTTP 头部让浏览器与服务器进行沟通，从而决定请求或响应是应该成功，还是应该失败。CORS是跨源AJAX请求的根本解决方法。

CORS需要浏览器和服务器同时支持。目前，所有浏览器都支持该功能，IE浏览器不能低于IE10。

整个CORS通信过程，都是浏览器自动完成，不需要用户参与。对于开发者来说，CORS通信与同源的AJAX通信没有差别，代码完全一样。浏览器一旦发现AJAX请求跨源，就会自动添加一些附加的头信息，有时还会多出一次附加的请求，但用户不会有感觉。

因此，实现CORS通信的关键是服务器。**只要服务器实现了CORS接口**，就可以跨源通信。

浏览器将CORS请求分成两类：简单请求（simple request）和非简单请求（not-so-simple request）。

#### 简单请求

只要同时满足以下两大条件，就属于简单请求。

（1) 请求方法是以下三种方法之一：`HEAD`，`GET`，`POST`

（2）HTTP的头信息不超出以下几种字段：

    Accept
    Accept-Language
    Content-Language
    Last-Event-ID
    Content-Type：只限于三个值application/x-www-form-urlencoded、multipart/form-data、text/plain

对于简单请求，浏览器直接发出CORS请求。具体来说，就是在头信息之中，增加一个**Origin**字段：
```
Origin: http://www.example.com
```
上面的头信息中，Origin字段用来说明，本次请求来自哪个源（协议 + 域名 + 端口）。服务器根据这个值，决定是否同意这次请求。

如果Origin指定的源，不在许可范围内，服务器会返回一个正常的HTTP回应。浏览器发现，这个回应的头信息没有包含Access-Control-Allow-Origin字段，就知道出错了，从而抛出一个错误，被XMLHttpRequest的onerror回调函数捕获。注意，这种错误无法通过状态码识别，因为HTTP回应的状态码有可能是200。

如果Origin指定的域名在许可范围内，服务器返回的响应，会多出几个头信息字段：

    Access-Control-Allow-Origin: http://www.example.com
    Access-Control-Allow-Credentials: true
    Access-Control-Expose-Headers: FooBar

**Access-Control-Allow-Origin**

该字段是必须的。它的值要么是请求时Origin字段的值，要么是一个`*`，表示接受任意域名的请求。

**Access-Control-Allow-Credentials**

该字段可选。它的值是一个布尔值，表示是否允许发送Cookie。默认情况下，Cookie不包括在CORS请求之中。设为`true`，即表示服务器明确许可，Cookie可以包含在请求中，一起发给服务器。这个值也只能设为true，如果服务器不要浏览器发送Cookie，删除该字段即可。

**Access-Control-Expose-Headers**

该字段可选。CORS请求时，`XMLHttpRequest`对象的`getResponseHeader`方法只能拿到**6**个基本字段：`Cache-Control`、`Content-Language`、`Content-Type`、`Expires`、`Last-Modified`、`Pragma`。如果想拿到其他字段，就必须在`Access-Control-Expose-Headers`里面指定。上面的例子指定，`getResponseHeader('FooBar')`可以返回`FooBar`字段的值。

#### withCredentials 属性
CORS请求默认不发送凭证(Cookie、HTTP认证及客户端SSL证明等)。

如果要把Cookie发到服务器，一方面要服务器同意：

    Access-Control-Allow-Credentials: true

另一方面，开发者必须在AJAX请求中打开withCredentials属性。

    var xhr = new XMLHttpRequest();
    xhr.withCredentials = true;

否则，即使服务器同意发送Cookie，浏览器也不会发送。或者，服务器要求设置Cookie，浏览器也不会处理。

但是，如果省略withCredentials设置，有的浏览器还是会一起发送Cookie。这时，可以显式关闭withCredentials。

    xhr.withCredentials = false;

需要注意的是，如果要发送Cookie，`Access-Control-Allow-Origin`就不能设为星号，必须指定明确的、与请求网页一致的域名。同时，Cookie**依然遵循同源策略**，只有用服务器域名设置的Cookie才会上传，其他域名的Cookie并不会上传，且（跨源）原网页代码中的`document.cookie`也无法读取服务器域名下的Cookie。

#### 非简单请求

非简单请求是那种对服务器有特殊要求的请求，比如请求方法是`PUT`或`DELETE`，或者`Content-Type`字段的类型是`application/json`。

非简单请求的CORS请求，会在正式通信之前，增加一次HTTP查询请求，称为**预检**请求（preflight）。

浏览器先询问服务器，当前网页所在的域名是否在服务器的许可名单之中，以及可以使用哪些HTTP动词和头信息字段。只有得到肯定答复，浏览器才会发出正式的XMLHttpRequest请求，否则就报错。

下面是一段浏览器的JavaScript脚本。

    var url = 'http://www.example.com';
    var xhr = new XMLHttpRequest();
    xhr.open('PUT', url, true);
    xhr.setRequestHeader('X-Custom-Header', 'value');
    xhr.send();

上面代码中，HTTP请求的方法是`PUT`，并且发送一个自定义头信息`X-Custom-Header`。

浏览器发现，这是一个非简单请求，就自动发出一个**预检**请求，要求服务器确认可以这样请求。下面是这个"预检"请求的HTTP头信息。

    OPTIONS /cors HTTP/1.1
    Origin: http://www.example.com
	Access-Control-Request-Method: PUT
	Access-Control-Request-Headers: X-Custom-Header
	Host: api.alice.com
	Accept-Language: en-US
	Connection: keep-alive
	User-Agent: Mozilla/5.0...

预检请求用的请求方法是**OPTIONS**，表示这个请求是用来询问的。头信息里面，关键字段是`Origin`，表示请求来自哪个源。

除了Origin字段，预检请求的头信息包括两个特殊字段。

`Access-Control-Request-Method`

该字段是必须的，用来列出浏览器的CORS请求会用到哪些HTTP方法，上例是PUT。

`Access-Control-Request-Headers`

该字段是一个逗号分隔的字符串，指定浏览器CORS请求会额外发送的头信息字段，上例是`X-Custom-Header`。

服务器收到"预检"请求以后，检查了`Origin`、`Access-Control-Request-Method`和`Access-Control-Request-Headers`字段以后，确认允许跨源请求，就可以做出回应。
```javascript
HTTP/1.1 200 OK
Date: Mon, 01 Dec 2008 01:15:39 GMT
Server: Apache/2.0.61 (Unix)
Access-Control-Allow-Origin: http://www.example.com
Access-Control-Allow-Methods: GET, POST, PUT
Access-Control-Allow-Headers: X-Custom-Header
Content-Type: text/html; charset=utf-8
Content-Encoding: gzip
Content-Length: 0
Keep-Alive: timeout=2, max=100
Connection: Keep-Alive
Content-Type: text/plain
```
上面的HTTP回应中，关键的是`Access-Control-Allow-Origin`字段，表示`http://www.example.com`可以请求数据。该字段也可以设为星号，表示同意任意跨源请求。

如果浏览器否定了预检请求，会返回一个正常的HTTP回应，但是没有任何CORS相关的头信息字段。这时，浏览器就会认定，服务器不同意预检请求，因此触发一个错误，被XMLHttpRequest对象的onerror回调函数捕获。控制台会打印出如下的报错信息。

    XMLHttpRequest cannot load http://www.example.com.
    Origin http://www.example.com is not allowed by Access-Control-Allow-Origin.

服务器回应的其他CORS相关字段如下。

`Access-Control-Allow-Methods`

该字段必需，它的值是逗号分隔的一个字符串，表明服务器支持的所有跨域请求的方法。注意，返回的是所有支持的方法，而不单是浏览器请求的那个方法。这是为了避免多次"预检"请求。

`Access-Control-Allow-Headers`

如果浏览器请求包括`Access-Control-Request-Headers`字段，则`Access-Control-Allow-Headers`字段是必需的。它也是一个逗号分隔的字符串，表明服务器支持的所有头信息字段，不限于浏览器在"预检"中请求的字段。

`Access-Control-Allow-Credentials`

该字段与简单请求时的含义相同。

`Access-Control-Max-Age`

该字段可选，用来指定本次预检请求的有效期，单位为秒。在此期间，不用发出另一条预检请求。

一旦服务器通过了预检请求，以后每次浏览器正常的CORS请求，就都跟简单请求一样，会有一个`Origin`头信息字段。服务器的回应，也都会有一个`Access-Control-Allow-Origin`头信息字段。

下面是预检请求之后，浏览器的正常CORS请求:
```javascript
PUT /cors HTTP/1.1
Origin: http://www.example.com
Host: api.alice.com
X-Custom-Header: value
Accept-Language: en-US
Connection: keep-alive
User-Agent: Mozilla/5.0...
```
上面头信息的Origin字段是浏览器自动添加的:

下面是服务器正常的回应。

    Access-Control-Allow-Origin: http://www.example.com
    Content-Type: text/html; charset=utf-8

上面头信息中，Access-Control-Allow-Origin字段是每次回应都必定包含的。

### 其它跨域技术

在CORS出现以前，要实现跨域 Ajax 通信颇费一些周折。开发人员想出了一些办法，利用 DOM 中能够执行跨域请求的功能，在不依赖 XHR 对象的情况下也能发送某种请求。 

#### 图像Ping

使用`<img>`标签，图像是可以跨域的。图像Ping是与服务器进行简单、单向的跨域通讯的一种方式。请求的数据通过查询字符串，响应可以是任意内容，但通常是像素图或204响应。浏览器通过侦听load和error事件，能知道响应什么时候接收到。
```javascript
var img = new Image(); 
img.onload = img.onerror = function() { 
  alert(" Done!");
}; 
img.src = "http://www.example.com/test?name=Nicholas";
```
图像Ping最常用于跟踪用户点击页面或动态广告曝光次数。缺点是只能发送GET请求，二是无法访问服务器的响应文本。

#### JSONP

JSONP是JSON with padding的简写，是应用JSON的新方法。JSONP由两部分组成：回调函数和数据。回调函数名称一般在请求中指定，数据是传入回调函数的JSON数据：
```javascript
function handleResponse(response) { 
  alert(" You’ re at IP address " + response.ip + ", which is in " +response.city + ", " + response.region_name); 
} 

var script = document.createElement("script"); 
script.src = "http://freegeoip.net/json/?callback=handleResponse"; 
document.body.insertBefore(script, document.body.firstChild);
```
JSONP能够直接响应文本，支持双向通讯。两点不足是：从其它域中加载代码不安全，要确定JSONP请求是否失败并不容易。

#### Comet
Comet是一种从服务器向页面推送数据的技术。

有两种实现Comet的方式：长轮询和流。

长轮询：页面发起一个请求，服务器一直保持连接打开，直到有数据发送。发送完数据后，浏览器关闭连接，随即再发起新请求。使用XHR和setTimeout就能实现。

HTTP流：浏览器向服务器发送一个请求，而服务器保持连接打开，然后周期性的向浏览器发送数据。

所有服务器端语言都支持打印到输出缓存然后刷新（ 将输出缓存中的内容 一次性全部发送到客户端）的功能。 而这正是实现 HTTP 流的关键所在。

通过侦听 readystatechange 事件及检测 readyState 的值是否为 3， 就可以利用 XHR 对象实现 HTTP 流。

Comet连接很容易出错，为此又创建了两个新的接口。

#### SSE
SSE，服务器发送事件

#### Web Socket
WebSocket是一种通信协议，使用`ws://`（非加密）和`wss://`（加密）作为协议前缀。该协议不实行同源政策，只要服务器支持，就可以通过它进行跨源通信。
```javascript
var socket = new WebSocket(" ws://www.example.com/server.php");
```
下面是一个例子，浏览器发出的WebSocket请求的头信息：
```javascript
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
Origin: http://example.com
```
上面代码中，有一个字段是Origin，表示该请求的请求源（origin），即发自哪个域名。

正是因为有了Origin这个字段，所以WebSocket才没有实行同源政策。因为服务器可以根据这个字段，判断是否许可本次通信。如果该域名在白名单内，服务器就会做出如下回应。
```javascript
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: HSmrc0sMlYUkAGmm5OPpG2HaGWk=
Sec-WebSocket-Protocol: chat
```