---
title: PWN-College-Web-Writeup
date: 2024-06-27 11:02:00
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

https://pwn.college/intro-to-cybersecurity/talking-web/

## 有价值的问题

### 1. 工具使用

`curl`工具的用法

- -X 指定请求方法POST GET
- -L 追踪重定向之后的url
- -v 显示详细的请求内容
- -i 输出请求头和响应体
- -H 传递自定义的请求头
- -d 传输的参数或数据
- -c 保存网页传递过来的cookie
- -b 读取文件中的cookie

`nc`

- 不支持自动重定向，需要手动检查
- -l  是 **listen（监听）** 的缩写，表示让 Netcat 进入监听模式，等待连接进来。
- -v verbose 使用这个选项后，Netcat 会在终端上打印出更多的连接细节，比如连接状态、IP 地址、端口等信息。
- `--no-keepalive`，请求结束后立即关闭链接。

```
-l 用于指定nc将处于侦听模式。指定该参数，则意味着nc被当作server，侦听并接受连接，而非向其它地址发起连接。
-p <port> 暂未用到（老版本的nc可能需要在端口号前加-p参数，下面测试环境是centos6.6，nc版本是nc-1.84，未用到-p参数）
-s 指定发送数据的源IP地址，适用于多网卡机
-u 指定nc使用UDP协议，默认为TCP
-v 输出交互或出错信息，新手调试时尤为有用
-w 超时秒数，后面跟数字
-z 表示zero，表示扫描时不发送任何数据
```

### 2. **服务器如何区分http包是由nc工具发送的，还是由python request库发送的呢？**

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

4. echo -e 输出转义字符

  转义字符指用反斜线字符“\”作为转义字符，来表示那些不可打印的ASCII控制符。譬如 `\r \n`这些



### 3. nc和curl工具的区别

`nc` (Netcat) 和 `curl` 是两种不同的网络工具，虽然它们都可以用于网络通信，但它们的功能和用途有显著的区别。

#### 1. **用途和功能**

- **`nc` (Netcat):**
  - **General-purpose network tool**: `nc` 是一个通用的网络工具，主要用于读写网络连接。它可以在 TCP 或 UDP 协议上工作，并且能够监听端口、创建反向 shell、传输文件，甚至可以用作简单的服务器。
  - **Low-level operations**: `nc` 更接近于底层，它不会自动处理 HTTP 协议细节（如重定向、编码、解码），而是允许你直接发送和接收原始的网络数据包。
  - **No built-in HTTP support**: 虽然可以用 `nc` 发送 HTTP 请求，但它不会自动生成 HTTP 头，也不会解析服务器的响应。
- **`curl`:**
  - **Dedicated for HTTP and other protocols**: `curl` 是一个专门用于与各种协议（如 HTTP, HTTPS, FTP, etc.）进行交互的工具。它主要用于下载文件、发送 HTTP 请求、处理 API 调用等。
  - **High-level operations**: `curl` 封装了大量的高层操作，如自动跟随重定向、处理 cookies、支持表单数据提交、多种身份验证方法、SSL/TLS 支持等。它使得与 HTTP 协议交互变得非常简单。
  - **Built-in support for many protocols**: `curl` 原生支持多种网络协议，不仅限于 HTTP/HTTPS。

#### 2. **常见用途**

- **`nc` (Netcat):**
  - **Port scanning**: 可以用来扫描开放端口。
  - **Simple server**: 通过监听指定端口，`nc` 可以作为一个简单的服务器来接收数据。
  - **Reverse shell**: 在网络安全中，`nc` 经常用于创建反向 shell 以便于访问远程主机。
  - **File transfer**: 可以用来在两台主机之间传输文件。
- **`curl`:**
  - **HTTP requests**: 用于发送各种类型的 HTTP 请求（GET, POST, PUT, DELETE 等）并处理响应。
  - **File download/upload**: 通过 HTTP, FTP 等协议下载或上传文件。
  - **API interaction**: 常用于与 RESTful APIs 进行交互，发送 JSON 数据，接收 JSON 响应。
  - **Web scraping**: 在需要时可以通过 `curl` 来获取网页内容。

#### 3. **协议支持**

- **`nc` (Netcat):**
  - **Protocols**: 支持 TCP 和 UDP 传输，但没有内置对高级协议（如 HTTP, FTP, SSL/TLS）的支持。
- **`curl`:**
  - **Protocols**: 支持多种高级协议，如 HTTP, HTTPS, FTP, FTPS, SCP, SFTP, SMTP, POP3, IMAP, SMB, TELNET, LDAP 等。

#### 4. **使用难度**

- **`nc` (Netcat):**
  - **Learning curve**: 由于它的低级别和通用性，`nc` 需要用户对网络通信有较深的理解。它适合用于调试、网络探索或安全研究。
  - **Manual operation**: 你需要手动构造请求、解析响应。
- **`curl`:**
  - **Ease of use**: `curl` 更加简单易用，尤其是处理 HTTP/HTTPS 请求时。大部分功能已经内置，你只需要使用相应的选项即可。
  - **Automated operations**: 它自动处理了许多 HTTP 细节，如编码、重定向、身份验证等。

#### 5. **示例对比**

- **使用 `nc` 发送 HTTP 请求:**

  ```bash
  echo -e "GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n" | nc example.com 80
  ```

- **使用 `curl` 发送 HTTP 请求:**

  ```bash
  curl http://example.com
  ```

#### 总结

- **`nc`** 是一个功能强大的通用网络工具，适合低级网络任务，如端口扫描、简单服务器设置、文件传输等。
- **`curl`** 是一个专注于处理多种网络协议的高层工具，特别擅长处理 HTTP 请求，适用于下载文件、与API交互等任务。

`nc` 更加灵活但需要更多的手动操作，而 `curl` 更加简单且自动化，适合处理标准协议和任务。

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

```bash
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

## level 3

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

## level 4

这次必须要用curl发。

```python
hacker@talking-web~level4:~$ curl -H "Host: a9b5812fe3df16d707b6f1c5a0c165be" http://127.0.0.1:80
pwn.college{gtnEkBK70at3p5XBaaJ-MHbZcrD.dFzNyMDL0czNxEzW}
```

## level 8

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

## level 10

```bash
curl http://127.0.0.1//ea218bfd%20a7e1e4ff/cc048c5a%201da3f711
```

构造URL时需要将空格替换为正确的URL编码每个需要编码的字符都被替换为 `%` 加上该字符的两位十六进制表示。例如，空格（）编码为 `%20`，字符 `#` 编码为 `%23`。

## level 11

同上，要用nc发

```bash
echo -e "GET 路由 HTTP/1.1\r\nHost:127.0.0.1\r\n\r\n" | nc 127.0.0.1 80
```

## level 16

```bash
curl "http://127.0.0.1:80/?a=66799705a636a300af5e48b8561fc084&b=5e5c813c%20f8bc26de%2626bd8630%23d7a6e240"
```

- 因为在命令行中，`&`代表的含义是在后台执行命令，所以为了避免歧义，需要将url用双引号包含起来。
- 然后在url传递参数的过程中，因为 `#`、`&`、`?`等有特别含义，所以需要进行编码

![20240903103816](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20240903103816.png)

## level 19

`curl http://***.***.**.**/api/api -X POST -d "parameterName1=parameterValue1&parameterName2=parameterValue2"`

`curl http://***.***.**.**/api/api -X POST -H "Content-Type:application/json" -d '{"parameterName1":"parameterValue1","parameterName2":"parameterValue2"}'`

这两种请求形式，http数据包有区别吗

1. Content-Type:

第一种请求：没有指定 `Content-Type`，默认的 `Content-Type` 为 `application/x-www-form-urlencoded`。这意味着数据将被编码为表单格式，即 `parameterName1=parameterValue1&parameterName2=parameterValue2 `这样的键值对。
第二种请求：明确指定了 `Content-Type: application/json`，表示请求体中的数据是JSON格式，即 `{"parameterName1":"parameterValue1","parameterName2":"parameterValue2"}`。

2. 数据编码:
   第一种请求的数据在HTTP包中会以URL编码形式发送，类似表单提交。服务器通常会解析这种数据为键值对。
   第二种请求的数据以JSON格式的纯文本发送，服务器会将其解析为JSON对象。
3. 服务器解析:
   在第一种情况下，服务器会将数据解析为普通的表单数据 `（application/x-www-form-urlencoded）`，一般用于简单的键值对数据传递。
   在第二种情况下，服务器会期望解析JSON格式的数据，用于传递更复杂的结构化数据。

## level 20

这个 `Content-Length`的长度必须要对，不然会多加入 `\r`和 `\n`，或者少了一些字母。

```bash
hacker@talking-web~level20:~$ echo -e "POST / HTTP/1.0\r\nHost:127.0.0.1\r\nContent-Type:application/x-www-form-urlencoded\r\nContent-Length:34\r\n\r\na=3cd954dd825f6a417a2b4f91e739fcd7\r\n" | nc 127.0.0.1 80
HTTP/1.1 200 OK
Server: Werkzeug/3.0.3 Python/3.8.10
Date: Tue, 03 Sep 2024 03:11:43 GMT
Content-Length: 58
Server: pwn.college
Connection: close

pwn.college{Y4AFDukL3sK4hTBApxLHWCwphbD.ddDOyMDL0czNxEzW}

```

## level 29

这种形式下不需要转义&，是因为传参不再是拼接到url里面进行传参了。

至于为什么最后一行的 `"`需要用 `\"`是因为，想让shell将其当作字符串进行处理，而不是将其视为是字符串的结束标志。

```bash
echo -e "POST / HTTP/1.1\r\n\
Host: 127.0.0.1\r\n\
Content-Type: application/json\r\n\
Content-Length: $(echo -n '{"a":"723dea0ce0b6086f8767604c36f07fd8","b":{"c":"a5755b0f","d":["1de2f1a7","4a11da76 a6130af5&1bd198c3#6fd30fed"]}}' | wc -c)\r\n\
Connection: close\r\n\
\r\n\
{\"a\":\"723dea0ce0b6086f8767604c36f07fd8\",\"b\":{\"c\":\"a5755b0f\",\"d\":[\"1de2f1a7\",\"4a11da76 a6130af5&1bd198c3#6fd30fed\"]}}" | nc 127.0.0.1 80

```

## level 34

```bash
curl -c cookies.txt http://127.0.0.1:80 && curl -b cookies.txt http://127.0.0.1:80 
curl -L -H "Cookie: cookie=6bdc50bc20a6b6dcd03710fd1b5db9a9" http://127.0.0.1:80
```
