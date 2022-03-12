---
title: 深入分析 Session & Cookie
date: 2018-09-22 23:04:01
tags: Https
categories: 服务端开发
---

因为Http协议是无状态的，服务端不知道用户上一次做了什么操作或者请求了什么接口。简单来说就是浏览器发送了两次请求，服务端是无法识别这两次请求是否来自同一个用户。这也是为什么会有 `Session` 和 `Cookie` 的原因，它们的作用就是用来维持客户端与服务端的会话。更简单的说，就是让服务端知道每一个请求都是来自于哪个用户。

<!-- more -->

### Cookie 是什么

简单的说，Cookie 就是浏览器储存在用户电脑上的一小段文本文件。Cookie 是纯文本格式，不包含任何可执行的代码。Cookie 总是被保存在客户端，按照客户端中的存储位置，可以分为内存 Cookie 和 硬盘 Cookie。内存 Cookie 由浏览器进行维护，保存在内存中。浏览器一旦关闭后就会消失。硬盘 Cookie 保存在硬盘中，会有一个过期时间。除非用户手动清理了 Cookie 或者到了过期时间，硬盘 Cookie 存在的时间比较长。所以按存在时间，还可以分为持久化 Cookie 和 非持久化 Cookie。

### 创建 Cookie
Web Server 通过发送一个 `set-Cookie` 的 `响应头` 来创建一个 `Cookie`，比如在 Servlet 中我们可以这样做：
```java
@WebServlet("/cookies")
public class CookiesController extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) {
        String uuid = UUID.randomUUID().toString().replace("-", "");
        Cookie cookieExample = new Cookie("USER_ID", uuid);
        response.addCookie(cookieExample);
    }

}
```
之后启动对应的服务，我们请求一下这个接口，观察一下响应头。
```console
$ curl -i http://localhost:8000/cookies-example/cookies
HTTP/1.1 200 OK
Server: Apache-Coyote/1.1
Set-Cookie: USER_ID=f304ec6313cc459abd9a24f543644c5c # 这个便是我们设置的 Cookie
Content-Length: 0
Date: Thu, 20 Sep 2018 13:38:43 GMT
```

### Cookie 的过期时间
Web Server 通过设置 MaxAge 的值来控制 Cookie 的过期时间。如果 MaxAge 属性为正数，则表示该 Cookie 会在 MaxAge 秒之后自动失效。浏览器会将 MaxAge 为正数的 Cookie 持久化，即写到对应的 Cookie 文件中。无论客户关闭了浏览器还是电脑，只要还在 MaxAge 秒之前，登录网站时该 Cookie 仍然有效。如果 MaxAge 为负数，则表示该 Cookie 仅在本浏览器窗口以及本窗口打开的子窗口内有效，关闭窗口后该 Cookie 即失效。MaxAge 为负数的 Cookie，为临时性 Cookie，不会被持久化，不会被写到 Cookie 文件中。Cookie 信息保存在浏览器内存中，因此关闭浏览器该 Cookie 就消失了。Cookie 默认的 MaxAge 值为-1。下面这段代码将设置 Cookie 一天后过期。
```java
cookieExample.setMaxAge(24 * 60 * 60);
```
```console
$ curl -i http://localhost:8000/cookies-example/cookies
HTTP/1.1 200 OK
Server: Apache-Coyote/1.1
# Expires 字段就是这个 Cookie 的到期时间
Set-Cookie: USER_ID=dbc1ab22f88343a7b16e86507f773ee3; Expires=Fri, 21-Sep-2018 13:41:37 GMT
Content-Length: 0
Date: Thu, 20 Sep 2018 13:41:37 GMT
```

### Cookie 的域名设置
Cookie 具有不可跨域名性。根据 Cookie 规范，浏览器访问 `https://www.google.com` 只会携带 Google 的 Cookie，而不会携带 `https://www.baidu.com` 的 Cookie。Google 也只能操作 Google 的 Cookie，而不能操作 Baidu 的 Cookie。对于子域名来讲 Cookie 无法设置当前子域名和其父域名之外的其他域名。比如你 **当前网站的域名** 或 **指定映射的域名** 为 `bar.baidu.com`，那么 Cookie 的 domain 仅能设置为 `bar.baidu.com` 或其父域名 `baidu.com`，其他的域名即使设置了，也一律无效。
```java
cookieExample.setDomain("example.com");
```
```console
$ curl -i http://localhost:8000/cookies-example/cookies
HTTP/1.1 200 OK
Server: Apache-Coyote/1.1
# Domain 字段就是 Cookie 指定的域名
Set-Cookie: USER_ID=4d36ccfe3dca47a0a8c0bc1354ce4dfb; Domain=example.com
Content-Length: 0
Date: Thu, 20 Sep 2018 14:00:40 GMT
```

### HttpOnly 选线
微软的 IE6 SP1 在 Cookie 中引入了一个新的选项：`HttpOnly` 背后的意思是告之浏览器该 Cookie 绝不能通过 `JavaScript` 的 `document.cookie` 属性访问。换句话说，设置了 `HttpOnly` 的 Cookie 只能在服务端进行修改，浏览器是不循序修改的。设计该特征意在提供一个安全措施来帮助阻止通过 JavaScript 发起的跨站脚本攻击 (XSS) 窃取 Cookie 的行为。一旦设定这个标记，通过 `documen.coookie` 则不能再访问该 Cookie。IE 同时更近一步并且不允许通过 `XMLHttpRequest` 的 `getAllResponseHeaders()` 或 `getResponseHeader()` 方法访问 Cookie，然而仍然有一些浏览器则允许此行为。
```java
cookieExample.setHttpOnly(true);
```
```console
$ curl -i http://localhost:8000/cookies-example/cookies
HTTP/1.1 200 OK
Server: Apache-Coyote/1.1
# 禁止浏览器修改 Cookie 的标记
Set-Cookie: USER_ID=0672f1bfca154a09aae93054fa2dc1d6; HttpOnly
Content-Length: 0
Date: Thu, 20 Sep 2018 14:23:02 GMT
```

### Secure 选项
默认情况下，在 Https 链接上传输的 Cookie 都会被自动添加上 `secure` 选项。只有当一个请求通过 SSL 或 Https 创建时，包含 secure 选项的 cookie 才能被发送至服务器。

### 使用 Cookie 的限制
Cookie 存在许多限制条件，来阻止 Cookie 滥用并保护浏览器和服务器免受一些负面影响。有两种 Cookie 限制条件：Cookie 的属性和 Cookie 的总大小。原始规范中限定每个域名下不超过 20 个 Cookie，早期的浏览器都遵循该规范，并且在 IE7 中有更近一步的提升。在微软的一次更新中，他们在 IE7 中增加 Cookie 的限制数量到 50 个，与此同时 Opera 限定 Cookie 数量为 30 个，Safari 和 Chrome 对与每个域名下的 Cookie 个数没有限制。发向服务器的所有 Cookie 的最大数量（空间）仍旧维持原始规范中所指出的：4KB。所有超出该限制的 Cookie 都会被截掉并且不会发送至服务器。

### Cookie 的自动删除
Cookie 是会被浏览器自动删除的，但不外乎以下几个原因：
* 会话 Cookie (Session Cookie) 在会话结束时（浏览器关闭）会被删除
* 持久化 Cookie（Persistent Cookie）在到达失效日期时会被删除
* 如果浏览器中的 Cookie 数量达到限制，那么 Cookie 会被删除以为新建的 Cookie 创建空间。

### 客户端操作 Cookie
在 JavaScript 中通过 `document.cookie` 属性，你可以创建、维护和删除 Cookie。创建 Cookie 时该属性等同于 Set-Cookie 消息头，而在读取 Cookie 时则等同于 Cookie 消息头。在创建一个 Cookie 时，你需要使用和 Set-Cookie 期望格式相同的字符串，例如：
```JavaScript
document.cookie = "USER_ID=0672f1bfca154a09aae93054fa2dc1d6; domain=example.com"
```

### 总结
为了高效的利用 Cookie，仍旧有许多要了解和弄明白的东西。对于一项创建于十多年前但仍旧如最初实现的那样被使用至今的技术来说，这是件多不可思议的事。本篇只是提供了一些每个人都应该知道的关于浏览器 Cookie 的基本指导，但无论如何，也不是一个完整的参考。对于今天的 Web 来说 Cookie 仍旧起着非常重要的作用，并且不恰当的管理 Cookie 会导致各种各样的问题，从最糟糕的用户体验到安全漏洞。参考文档：

- [https://zh.wikipedia.org/wiki/Cookie](https://zh.wikipedia.org/wiki/Cookie)
- [https://humanwhocodes.com/blog/2009/05/05/http-cookies-explained/](https://humanwhocodes.com/blog/2009/05/05/http-cookies-explained/)
