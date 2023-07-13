# Cookie

`Cookie` 曾一度用于客户端数据的存储，因当时并没有其他合适的存储办法而作为唯一的存储手段，但现在推荐使用现代存储 API。由于服务器指定 `Cookie` 后，浏览器的每次请求都会携带 `Cookie` 数据，会带来额外的性能开销（尤其是在移动环境下）。新的浏览器 API 已经允许开发者直接将数据存储到本地，如使用 `Web storage API（localStorage 和 sessionStorage`或 `IndexedDB` 。

## Set-Cookie

响应标头`Set-Cookie`被用来由服务器端向用户代理发送cookie，所以用户代理可在后续的请求中将其发送回服务器。服务器要发送多个cookie，则应该在同一响应中发送多个`Set-Cookie` 标头。

### 语法

```HTML
Set-Cookie: <cookie-name>=<cookie-value>
Set-Cookie: <cookie-name>=<cookie-value>; Expires=<date>
Set-Cookie: <cookie-name>=<cookie-value>; Max-Age=<number>
Set-Cookie: <cookie-name>=<cookie-value>; Domain=<domain-value>
Set-Cookie: <cookie-name>=<cookie-value>; Path=<path-value>
Set-Cookie: <cookie-name>=<cookie-value>; Secure
Set-Cookie: <cookie-name>=<cookie-value>; HttpOnly

Set-Cookie: <cookie-name>=<cookie-value>; SameSite=Strict
Set-Cookie: <cookie-name>=<cookie-value>; SameSite=Lax
Set-Cookie: <cookie-name>=<cookie-value>; SameSite=None; Secure

// Multiple attributes are also possible, for example:
Set-Cookie: <cookie-name>=<cookie-value>; Domain=<domain-value>; Secure; HttpOnly
```

### 指令

`<cookie-name>=<cookie-value>`

名称/值对

`Expires=<date>`

可选。Cookie的生命周期。

如果没有设置这个属性，那么表示这是一个`session cookie`。一个`session`结束于客户端被关闭时，这意味着`session cookie`在彼时会被移除。

然而，很多浏览器支持会话恢复功能，这个功能可以使浏览器保留所有的`tab`标签，然后在重新打开浏览器的时候将其还原。

`Expires`属性，其截止时间与客户端相关，而非服务器的时间

`Max-Age=<number>`

可选。Cookie的生命周期。优先级高于`Expires`

在 `cookie` 失效之前需要经过的秒数。秒数为 0 或 -1 将会使 `cookie` 直接过期

`Domain=<domain-value>` 

可选, 指定 cookie 可以送达的主机名。

假如没有指定，那么默认值为当前访问地址中的Host部分（但是不包含子域名）。

与之前的规范不同的是，域名（.example.com）之前的点号会被忽略。

如果指定了一个域，则其子域也会被包含。如果设置 `Domain=mozilla.org`，则 Cookie 也包含在子域名中（如 `developer.mozilla.org`）

`Path=<path-value> `

可选, 指定一个 URL 路径，这个路径必须出现在要请求的资源的路径中才可以发送 Cookie 标头。

字符 `/` 可以解释为文件目录分隔符，此目录的下级目录也满足匹配的条件（例如，如果 `path=/docs`，那么

- `/docs`、`/docs/`、`/docs/Web/` 和 `/docs/Web/HTTP` 都满足匹配条件。
- `/`、`/docsets` 或者 `/fr/docs` 则不满足匹配条件。

`Secure `

可选, 一个带有安全属性的 `cookie` 只有在请求使用 `https:` 协议（`localhost` 不受此限制）的时候才会被发送到服务器。以阻止**中间人攻击**。

`HttpOnly`

可选, 用于阻止 `JavaScript` 通过 `Document.cookie` 属性访问 `cookie`。注意，设置了 HttpOnly 的 cookie 在 JavaScript 初始化的请求中仍然会被发送。例如，调用 XMLHttpRequest.send() 或 fetch()。其用于防范跨站脚本攻击（`XSS`）。

`SameSite=<samesite-value>`

可选, 允许服务器设定一则 `cookie` 不随着跨站请求一起发送，这样可以在一定程度上防范跨站请求伪造攻击（`CSRF`）。

可选的属性值有：

- `Strict`: 这意味浏览器仅对同一站点的请求发送 cookie，即请求来自设置 cookie 的站点。如果请求来自不同的域或协议（即使是相同域），则携带有 `SameSite=Strict` 属性的 cookie 将不会被发送。
- `Lax`: 这意味着 cookie 不会在跨站请求中被发送，如：加载图像或 frame 的请求。但 cookie 在用户从外部站点导航到源站时，cookie 也将被发送（例如，跟随一个链接）。这是 SameSite 属性未被设置时的默认行为。
- `None`: 这意味着浏览器会在跨站和同站请求中均发送 cookie。在设置这一属性值时，必须同时设置 Secure 属性，就像这样：`SameSite=None; Secure`。
备注： 与 SameSite Cookie 相关的标准作了如下变动：

SameSite 属性未被指定时，其默认行为是 `SameSite=Lax`。在过去，若未指定，所有的 cookie 均会被发送。

现在，携带 `SameSite=None` 属性的 cookie 必须同时设置 Secure 属性（换句话说，其仅能用于安全上下文）

来自同一域的 cookie 若使用了不同的协议（http: 或 HTTPS:），将不再被视为来自同一站点。

### Cookie 前缀
名称中包含 `__Secure-` 或 `__Host-` 前缀的 `cookie`，只可以应用在使用了安全连接（`HTTPS`）的域中，需要同时设置 `secure` 属性。

另外，假如 cookie 以 `__Host-` 为前缀，那么 `path` 属性的值必须为 `/`（表示整个站点），且不能含有 `Domain` 属性。

```HTML
// 当响应来自于一个安全域（HTTPS）的时候，二者都可以被客户端接受
Set-Cookie: __Secure-ID=123; Secure; Domain=example.com
Set-Cookie: __Host-ID=123; Secure; Path=/

// 缺少 Secure 指令，会被拒绝
Set-Cookie: __Secure-id=1

// 缺少 Path=/ 指令，会被拒绝
Set-Cookie: __Host-id=1; Secure

// 由于设置了 domain 属性，会被拒绝
Set-Cookie: __Host-id=1; Secure; Path=/; domain=example.com
```

## Document.cookie

JavaScript 通过 `Document.cookie` 访问 Cookie

读取所有可从此位置访问的 COOKIE
```JAVASCRIPT
allCookies = document.cookie;
```
在上面的代码中，`allCookies` 被赋值为一个字符串，该字符串包含所有的 `Cookie`，每条 cookie 以`; `分隔 (即， key=value 键值对)。

写Cookie
```JAVASCRIPT
document.cookie = "name=oeschger";
document.cookie = "favorite_food=tripe";
alert(document.cookie);
// 显示：name=oeschger;favorite_food=tripe

document.cookie = "test1=Hello";
document.cookie = "test2=World";

var myCookie = document.cookie.replace(/(?:(?:^|.*;\s*)test2\s*\=\s*([^;]*).*$)|^.*$/, "$1");

alert(myCookie);
// 显示：World
```

## 引用

[Cookie规范 rfc6265](https://datatracker.ietf.org/doc/html/rfc6265 "Cookie规范 rfc6265")