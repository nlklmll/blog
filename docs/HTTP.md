---
title: HTTP
date: 2026-07-15
tags: [web协议]
---

# HTTP



> [!IMPORTANT] 
>
> 学习资料[HTTP 概述 - HTTP | MDN](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Guides/Overview)

> [!TIP]
>
> 实用资料[HTTP 教程 | 菜鸟教程](https://www.runoob.com/http/http-tutorial.html)


---
## 1、典型的 HTTP 会话

在像 HTTP 这样的客户端——服务器（Client-Server）协议中，会话分为三个阶段：

1. 客户端建立一条 TCP 连接（如果传输层不是 TCP，也可以是其他适合的连接）。
2. 客户端发送请求并等待应答。
3. 服务器处理请求并送回应答，回应包括一个状态码和对应的数据。

从 HTTP/1.1 开始，连接在完成第三阶段后不再关闭，客户端可以再次发起新的请求。这意味着第二步和第三步可以连续进行数次。


---
## 2、[发送客户端请求](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Guides/Session#发送客户端请求)

访问 developer.mozilla.org（`https://developer.mozilla.org/`）的根页面，并告诉服务器用户代理倾向于该页面使用法语展示：

```
GET / HTTP/1.1
Host: developer.mozilla.org
Accept-Language: fr
```

注意最后的空行，它把标头与数据块分隔开。由于在 HTTP 标头中没有 `Content-Length`，数据块是空的，所以服务器可以在收到代表标头结束的空行后就开始处理请求。

例如，发送表单的结果：

```
POST /contact_form.php HTTP/1.1
Host: developer.mozilla.org
Content-Length: 64
Content-Type: application/x-www-form-urlencoded

name=Joe%20User&request=Send%20me%20one%20of%20your%20catalogue
```

HTTP 定义了一组[请求方法](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Reference/Methods)用来指定对目标资源的行为。它们一般是名词，但这些请求方法有时会被叫做 HTTP 动词。最常用的请求方法是 `GET` 和 `POST`：

- [`GET`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Reference/Methods/GET) 方法请求指定的资源。`GET` 请求应该只被用于获取数据。
- [`POST`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Reference/Methods/POST) 方法向服务器发送数据，因此会改变服务器状态。这个方法常在 [HTML 表单](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Forms)中使用。


---
## 3、[服务器响应结构](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Guides/Session#服务器响应结构)

当收到用户代理发送的请求后，Web 服务器就会处理它，并最终送回一个响应。与客户端请求很类似，服务器响应由一系列文本指令组成，并使用 CRLF 分隔，它们被划分为三个不同的块：

1. 第一行是*状态行*，包括使用的 HTTP 协议版本，然后是一个状态码（及其人类可读的描述文本）。
2. 接下来每一行都表示一个 HTTP 标头，为客户端提供关于所发送数据的一些信息（如类型、数据大小、使用的压缩算法、缓存指示）。与客户端请求的头部块类似，这些 HTTP 标头组成一个块，并以一个空行结束。
3. 最后一块是数据块，包含了响应的数据（如果有的话）。

```
HTTP/1.1 301 Moved Permanently
Server: Apache/2.4.37 (Red Hat)
Content-Type: text/html; charset=utf-8
Date: Thu, 06 Dec 2018 17:33:08 GMT
Location: https://developer.mozilla.org/ （目标资源的新地址，服务器期望用户代理去访问它）
Keep-Alive: timeout=15, max=98
Accept-Ranges: bytes
Via: Moz-Cache-zlb05
Connection: Keep-Alive
Content-Length: 325 (如果用户代理无法转到新地址，就显示一个默认页面)

<!DOCTYPE html>… (包含一个网站自定义页面，帮助用户找到丢失的资源)
```

> [!NOTE]
>
> 常用状态码

| 状态码      | 含义       | 渗透时遇到意味着             |
| ----------- | ---------- | ---------------------------- |
| **200**     | 成功       | 正常                         |
| **301/302** | 跳转       | 可能有跳转漏洞（开放重定向） |
| **403**     | 禁止访问   | 有权限控制，值得进一步测     |
| **404**     | 不存在     | 路径不对                     |
| **500**     | 服务器报错 | 可能暴露错误信息，可挖信息   |


---
## 4、HTTP Cookie

Cookie 主要用于以下三个方面：

- 会话状态管理

  如用户登录状态、购物车、游戏分数或其他需要记录的信息

- 个性化设置

  如用户自定义设置、主题和其他设置

- 浏览器行为跟踪

  如跟踪分析用户行为等

服务器收到 HTTP 请求后，服务器可以在响应标头里面添加一个或多个 [`Set-Cookie`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Reference/Headers/Set-Cookie) 选项。浏览器收到响应后通常会保存下 Cookie，并将其放在 HTTP [`Cookie`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Reference/Headers/Cookie) 标头内，向同一服务器发出请求时一起发送。你可以指定一个过期日期或者时间段之后，不能发送 cookie。你也可以对指定的域和路径设置额外的限制，以限制 cookie 发送的位置。

服务器使用 [`Set-Cookie`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Reference/Headers/Set-Cookie) 响应头部向用户代理（一般是浏览器）发送 Cookie 信息。一个简单的 Cookie 可能像这样：

```
Set-Cookie: <cookie-name>=<cookie-value>
```

这指示服务器发送标头告知客户端存储一对 cookie：

```
HTTP/1.0 200 OK
Content-type: text/html
Set-Cookie: yummy_cookie=choco
Set-Cookie: tasty_cookie=strawberry
```

现在，对该服务器发起的每一次新请求，浏览器都会将之前保存的 Cookie 信息通过 [`Cookie`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Reference/Headers/Cookie) 请求头部再发送给服务器。

```
GET /sample_page.html HTTP/1.1
Host: www.example.org
Cookie: yummy_cookie=choco; tasty_cookie=strawberry
```


---
## 5、HTTP 的重定向

在 HTTP 协议中，重定向操作由服务器向请求发送特殊的重定向响应而触发。重定向响应包含以 `3` 开头的[状态码](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Reference/Status)，以及 [`Location`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Reference/Headers/Location) 标头，其保存着重定向的 URL。

浏览器在接收到重定向时，它们会立刻加载 `Location` 标头中提供的新 URL。除了额外的往返操作中会有一小部分性能损失之外，重定向操作对于用户来说是不可见的。

不同类型的重定向映射可以划分为三个类别：

1. [永久重定向](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Guides/Redirections#永久重定向)(301、308)

   这种重定向操作是永久性的。它表示原 URL 不应再被使用，而选用新的 URL 替换它。搜索引擎机器人、RSS 阅读器以及其他爬虫将更新资源原始的 URL。

2. [临时重定向](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Guides/Redirections#临时重定向)(302、303、307)

   有时候请求的资源无法从其标准地址访问，但是却可以从另外的地方访问。在这种情况下，可以使用临时重定向。

   搜索引擎和其他爬虫不会记录新的、临时的 URL。在创建、更新或者删除资源的时候，临时重定向也可以用于显示临时性的进度页面。

3. [特殊重定向](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Guides/Redirections#特殊重定向)(300、304)

   [`304`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Reference/Status/304)（Not Modified）会使页面跳转到本地的缓存副本中（可能已过时），而 [`300`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Reference/Status/300)（Multiple Choice）则是一种手动重定向：将消息主体以 Web 页面形式呈现在浏览器中，列出了可能的重定向链接，用户可以从中进行选择。

指定重定向的其他方式

​	HTTP 重定向是创建重定向的最佳方式，但是有时候你并不能控制服务器。针对这些特定的应用情景，可以尝试在页面的head 中添加一个 /meta)元素，并将其 [`http-equiv`](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Reference/Elements/meta#http-equiv) 属性的值设置为 `refresh`。当显示页面的时候，浏览器会检测该元素，然后跳转到指定的页面。

```
<head>
  <meta http-equiv="Refresh" content="0; URL=http://example.com/" />
</head>
```

[`content`](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Reference/Global_attributes#content) 属性的值开头是一个数字，指示浏览器在等待该数字表示的秒数之后再进行跳转。建议始终将其设置为 `0` 来获取更好的无障碍体验。

​	在 JavaScript 中，重定向机制的原理是设置 [`window.location`](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/location) 的属性值，然后加载新的页面。

```
window.location = "https://example.com/";
```

与 HTML 重定向机制类似，这种方式并不适用于所有类型的资源，并且显然只有在执行 JavaScript 的客户端上才能使用。另外一方面，它也提供了更多的可能性：比如在只有满足了特定的条件的情况下才可以触发重定向机制的场景。