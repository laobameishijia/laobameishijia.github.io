---
title: PWN-College-Writeup
date: 2024-06-27 11:02:00
author: 美食家李老叭
img: 
top: false
hide: false
cover: false
coverImg: https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20240606170337.png
password: 
toc: true
mathjax: false
summary: CTF 
categories: CTF
tags:
    - 逆向
---
# pwn.college

https://pwn.college/intro-to-cybersecurity/talking-web/

## 有价值的问题

- **服务器如何区分http包是由nc工具发送的，还是由python request库发送的呢？**

**关键函数**

1. **`validate_client(name)` 函数**：

   - 这个函数是核心部分，它通过对比客户端进程的路径来确定请求是由哪个工具发出的。

2. **路径解析**：

   - ```
     correct_path = pathlib.Path(shutil.which(name, path=PATH)).resolve()
     ```

     - 通过 `shutil.which` 方法在系统 PATH 环境变量中查找工具的绝对路径，并解析为绝对路径。

   - ```
     server_connection
     ```

      和 

     ```
     client_connection
     ```

     - 使用 `psutil.Process().connections()` 和 `psutil.net_connections()` 获取服务器进程和客户端进程的连接信息。
     - 确定请求是从哪个客户端进程发出的。

3. **对比进程路径**：

   - ```
     client_process = psutil.Process(client_connection.pid)
     ```

     - 获取客户端进程的 PID，并创建 `psutil.Process` 对象。

   - ```
     client_path = pathlib.Path(client_process.exe())
     ```

     - 获取客户端进程的可执行文件路径。

   - ```
     validate("client", client_path, correct_path)
     ```

     - 验证客户端进程的路径是否与预期的路径（`curl` 或 `python`）匹配。

代码：

```python
def validate_client(name):
    PATH = "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    correct_path = pathlib.Path(shutil.which(name, path=PATH)).resolve()

    server_connection = next(
        connection
        for connection in psutil.Process().connections()
        if connection.fd == request.input_stream.fileno()
    )
    client_connection = next(
        connection
        for connection in psutil.net_connections()
        if connection.raddr == server_connection.laddr
        and connection.laddr == server_connection.raddr
    )
    client_process = psutil.Process(client_connection.pid)
    client_path = pathlib.Path(client_process.exe())

    validate("client", client_path, correct_path)

def validate(name, value, correct):
    assertion_message = f"Incorrect {name}: value `{value}`, should be `{correct}`\n"
    assert value == correct, assertion_message
    
```





## level2

使用nc构造一个http请求，发送到服务器就行了。

```bash
hacker@talking-web~level2:/challenge$ printf "GET / HTTP/1.0\r\n\r\n" | nc 127.0.0.1 80
HTTP/1.1 200 OK
Server: Werkzeug/3.0.3 Python/3.8.10
Date: Thu, 27 Jun 2024 02:57:20 GMT
Content-Length: 58
Server: pwn.college
Connection: close

pwn.college{QN4lchMdYBn2k2BOiFBvgK6IqMT.dljNyMDL0czNxEzW}
```

### 命令分解

```
swift
复制代码
printf "GET / HTTP/1.0\r\n\r\n" | nc host.example.com 80
```

1. **`printf "GET / HTTP/1.0\r\n\r\n"`**:

   - `printf` 是一个用于格式化输出的命令，它将字符串 `GET / HTTP/1.0\r\n\r\n` 输出到标准输出。

   - ```
     "GET / HTTP/1.0\r\n\r\n"
     ```

      是一条 HTTP GET 请求。这部分字符串分为以下几个部分：

     - `GET`：这是 HTTP 方法，表示获取资源。
     - `/`：这是请求的路径，在这个例子中是根路径，即首页。
     - `HTTP/1.0`：这是 HTTP 版本号。
     - `\r\n`：这是回车和换行符，表示 HTTP 头部的结束。
     - 再一次的 `\r\n`：表示 HTTP 请求头部的结束和请求体的开始。由于 GET 请求没有请求体，所以这部分就结束了整个请求。

2. **`|`**:

   - 管道符号 (`|`) 将前一个命令的输出传递给下一个命令作为输入。

3. **`nc host.example.com 80`**:

   - `nc` 是 Netcat 的缩写，是一个网络工具，可以用于读取和写入网络连接。
   - `host.example.com` 是目标主机名。
   - `80` 是目标端口号，HTTP 默认使用端口 80。

### 整体解释

该命令通过 `nc` 向指定的服务器 (`host.example.com`) 的端口 80 发送一个手工构建的 HTTP GET 请求。`printf` 命令生成的 HTTP 请求通过管道传递给 `nc`，`nc` 然后将这个请求发送给服务器。服务器处理请求后，会返回一个 HTTP 响应，该响应将显示在终端中。

## level3

这个要使用python requests库发送http包才能获取到flag

```python
import requests

response = requests.get('http://127.0.0.1:80')
print(response.text)
print(response.request.headers)
# pwn.college{E5LhFycGy6yYF4ln-T0gK8ZtcRU.dBzNyMDL0czNxEzW}
```

即便用nc模拟python的头，还是无法通过检测。

```bash
printf "GET / HTTP/1.1\r\nHost: 127.0.0.1\r\nUser-Agent: python-requests/2.25.1\r\nAccept-Encoding: gzip, deflate\r\nAccept: */*\r\nConnection: keep-alive\r\n\r\n" | nc 127.0.0.1 80
```

## level4

这次必须要用curl发。

```python
hacker@talking-web~level4:~$ curl -H "Host: a9b5812fe3df16d707b6f1c5a0c165be" http://127.0.0.1:80
pwn.college{gtnEkBK70at3p5XBaaJ-MHbZcrD.dFzNyMDL0czNxEzW}
```

## level8 

```bash
printf "GET /13d8cb22c3311435da99d6773d4dd96c HTTP/1.1\r\n\r\n" | nc 127.0.0.1 80
HTTP/1.1 200 OK
Server: Werkzeug/3.0.3 Python/3.8.10
Date: Thu, 27 Jun 2024 07:38:57 GMT
Content-Length: 58
Server: pwn.college
Connection: close

pwn.college{glFzdeCtyajxWzcz8Ei8kztjvxa.dVzNyMDL0czNxEzW}
```

## level10

```bash
curl http://127.0.0.1//ea218bfd%20a7e1e4ff/cc048c5a%201da3f711
```

构造URL时需要将空格替换为正确的URL编码每个需要编码的字符都被替换为 `%` 加上该字符的两位十六进制表示。例如，空格（` `）编码为 `%20`，字符 `#` 编码为 `%23`。



