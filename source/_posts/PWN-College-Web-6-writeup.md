---
title: PWN-College-Web-6-Writeup
date: 2024-09-14 10:46:00
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

https://pwn.college/intro-to-cybersecurity/cryptography/

## 有价值的问题

### 1. python 中int、hex、byte之间的转换

```python
a = 3735928559
print(a)# 3735928559
print(hex(a)) # 0xdeadbeef   int->转十六进制字符串
print(bytes.fromhex(hex(a)[2:])) # b'\xde\xad\xbe\xef' 十六进制字符串转 bytes
print((hex(a)[2:]).encode('ascii')) # b'deadbeef' 十六进制ASCII码字符串 转 bytes
print(int("deadbeef",16)) # 3735928559 十六进制字符串转 int
print(int.from_bytes(bytes.fromhex("deadbeef"), "big")) # 3735928559 十六进制字符串转 int
print(int.from_bytes(a.to_bytes(4), "big")) # 3735928559 字节  转 int
print(a.to_bytes(4)) # b'\xde\xad\xbe\xef'  int 转 字节数据
# 关于十六进制转字符串的问题
b'\xde\xad'  <==> b'ab'在形式上是一样的，只不过它通过ASCII码解码了一下，以后见到b'',里面不带\x的字母一律当将其转换成字母的ASCII码表示。
```
![20240917122217](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20240917122217.png)


### 2. pwntool的用法

`context.log_level = debug`，这个可以显示与进程交互的所有信息，全部输入输出的回显。



### 3. Base64 编码原理

https://juejin.cn/post/6994612829437296647


### 4. AES加密


AES（高级加密标准，Advanced Encryption Standard） 是一种对称加密算法。在对称加密中，加密和解密使用相同的密钥。发送方使用密钥将明文加密为密文，接收方则使用相同的密钥将密文解密回明文。因为密钥必须在加密和解密双方之间安全共享，所以对称加密要求双方拥有相同的密钥。

![20240916172338](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20240916172338.png)

### 5. 大小端序

大端序 (Big-Endian): 高位字节存储在内存的低地址处，低位字节存储在高地址处。即从左到右按值大小排列，像我们日常阅读的方式。
小端序 (Little-Endian): 低位字节存储在内存的低地址处，高位字节存储在高地址处。即从右到左按值大小排列。

```python
data = b'\x12\x34'
num_big = int.from_bytes(data, byteorder='big')
print(num_big)  # 输出: 4660 (即 0x1234)

num_big = int.from_bytes(data) # 默认大端序
print(num_big)  # 输出: 4660 (即 0x1234)

# 使用小端序
num_little = int.from_bytes(data, byteorder='little')
print(num_little)  # 输出: 13330 (即 0x3412)
```

### 6. python中==的含义

在 Python 中，`==` 是一个比较运算符，用于检查两个对象是否相等。它比较的是对象的值或内容，而不是对象的身份（即内存地址）。具体来说：

#### 对于基本数据类型

- **数值类型（如整数、浮点数）**：`==` 比较两个数值是否相等。例如，`5 == 5.0` 会返回 `True`，因为它们表示相同的数值。
- **字符串和字节串**：`==` 比较两个字符串或字节串的内容是否相同。注意，字符串和字节串是不同的类型，因此 `b'abc' == 'abc'` 会返回 `False`，因为一个是字节串，另一个是字符串。要比较它们，你需要将其中一个转换成另一个类型。  **尤其注意这个**

#### 对于复杂数据类型

- **列表、元组、集合、字典**：`==` 比较两个数据结构的内容是否相同。例如，`[1, 2, 3] == [1, 2, 3]` 会返回 `True`，而 `{'a': 1} == {'b': 1}` 会返回 `False`。

#### 对于对象

- **自定义对象**：`==` 比较两个对象的值是否相等，而不是它们是否是同一个对象。默认情况下，Python 会比较对象的 `id`（即内存地址），但可以通过定义类的 `__eq__` 方法来定制相等性比较。

#### 例子

##### 基本数据类型

```python
x = 10
y = 10
print(x == y)  # 输出: True

a = "hello"
b = "hello"
print(a == b)  # 输出: True
```

#### 自定义对象

```python
class Person:
    def __init__(self, name):
        self.name = name

    def __eq__(self, other):
        if isinstance(other, Person):
            return self.name == other.name
        return False

person1 = Person("Alice")
person2 = Person("Alice")
print(person1 == person2)  # 输出: True
```

#### 数据结构

```python
list1 = [1, 2, 3]
list2 = [1, 2, 3]
print(list1 == list2)  # 输出: True

dict1 = {'key': 'value'}
dict2 = {'key': 'value'}
print(dict1 == dict2)  # 输出: True
```

#### 总结

`==` 用于检查两个对象的值或内容是否相等，具体的比较逻辑可能会根据数据类型和自定义类的实现有所不同。




### 7. diffie-Hallman
diffie-Hallman

![diffie-Hallman](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20240923110157.png)


## level 5

通过flag文件的大小可以知道flag文件的长度是57。

解题思路就是：
1:
    63* A + p
    63* A + 猜测字母  ---- 这里只需要对比结果的前64个字节是否相等，就知道猜得到底对不对了。
2:
    62* A + pw
    62* A + p + 猜测字母

    ....
57:
    7 * A + {flag}
    7 * A + {flag - 1} + 猜测字母（最后的`}`） 

这样就能把flag的所有内容都解出来了。


​    


```python
import pwn
import base64 
chall= pwn.process("/challenge/run") 
chall.readuntil(b"secret ciphertext (b64): ") 
secret_ciphertext= chall.readline().decode() 
ciphertext_decoded=base64.b64decode(secret_ciphertext) 

def findNextChar(currFlag, numA): 
    chall.sendline(base64.b64encode(b"A"*numA)) 
    chall.readuntil("ciphertext (b64): ") 
    new_line=chall.readline()
    new_encryption=base64.b64decode(new_line)
    print(len(new_encryption))
    for i in range(32,128): 
        chall.sendline(base64.b64encode(b"A"*numA+ bytes(currFlag, 'ascii')+ bytes([i]))) 
        chall.readuntil("ciphertext (b64): ") 
        line=chall.readline()
        encryption=base64.b64decode(line) 
        if new_encryption[:64] == encryption[:64]:
            return chr(i)

flag = ""

for i in range(63, 6, -1):
    flag  = flag + findNextChar(flag, i)
print(flag)
print(len(flag))

```

## level 6

我把这一关的代码抽出来了，在解题的过程中，第一开始总是无法正确处理输入输出。

- B 接受的是 `int的十六进制字符串`(也就是字符串对应的ASCII码表示---字节流)
- 然后通过`int(name, 16)`将十六进制字符串转换为int类型。

所以你传输的也要是`十六进制字符串对应的字节流`

详情你可以看一下，问题1的讲解，总之一开始直接传输int类型所对应的字节表示，是错误的`譬如直接传输b'\xde\xad\xbe\xef'` ，B会按照字符串的形式对字节流进行解码，`\xde`这种字节按照`ASCII码`解码的话肯定是无法对应到正确的十六进制字母。所以会报错



``


```python
#!/bin/python

import sys
import string
import random
import pathlib
import base64
import json
import textwrap

from Crypto.Cipher import AES
from Crypto.Hash.SHA256 import SHA256Hash
from Crypto.PublicKey import RSA
from Crypto.Random import get_random_bytes
from Crypto.Random.random import getrandbits, randrange
from Crypto.Util.strxor import strxor
from Crypto.Util.Padding import pad, unpad


flag = open("/home/hacker/Web6/level6/flag", "rb").read()
# config = (pathlib.Path(__file__).parent / ".config").read_text()
level = 6


def show(name, value, *, b64=True):
    print(f"{name}: {value}")


def show_b64(name, value):
    show(f"{name} (b64)", base64.b64encode(value).decode())

def show_hex_block(name, value, byte_block_size=16):
    value_to_show = ""

    for i in range(0, len(value), byte_block_size):
        value_to_show += f"{value[i:i+byte_block_size].hex()}"
        value_to_show += " "
    show(f"{name} (hex)", value_to_show)


def show_hex(name, value):
    show(name, hex(value))


def input_(name):
    try:
        return input(f"{name}: ")
    except (KeyboardInterrupt, EOFError):
        print()
        exit(0)


def input_b64(name):
    data = input_(f"{name} (b64)")
    try:
        return base64.b64decode(data)
    except base64.binascii.Error:
        print(f"Failed to decode base64 input: {data!r}", file=sys.stderr)
        exit(1)


def input_hex(name):
    data = input_(name)
    try:
        return int(data, 16)
    except Exception:
        print(f"Failed to decode hex input: {data!r}", file=sys.stderr)
        exit(1)



def level6():
    """
    In this challenge you will perform a Diffie-Hellman key exchange.
    """
    # 2048-bit MODP Group from RFC3526
    p = int.from_bytes(bytes.fromhex(
        "FFFFFFFF FFFFFFFF C90FDAA2 2168C234 C4C6628B 80DC1CD1 "
        "29024E08 8A67CC74 020BBEA6 3B139B22 514A0879 8E3404DD "
        "EF9519B3 CD3A431B 302B0A6D F25F1437 4FE1356D 6D51C245 "
        "E485B576 625E7EC6 F44C42E9 A637ED6B 0BFF5CB6 F406B7ED "
        "EE386BFB 5A899FA5 AE9F2411 7C4B1FE6 49286651 ECE45B3D "
        "C2007CB8 A163BF05 98DA4836 1C55D39A 69163FA8 FD24CF5F "
        "83655D23 DCA3AD96 1C62F356 208552BB 9ED52907 7096966D "
        "670C354E 4ABC9804 F1746C08 CA18217C 32905E46 2E36CE3B "
        "E39E772C 180E8603 9B2783A2 EC07A28F B5C55DF0 6F4C52C9 "
        "DE2BCBF6 95581718 3995497C EA956AE5 15D22618 98FA0510 "
        "15728E5A 8AACAA68 FFFFFFFF FFFFFFFF"
    ), "big")
    g = 2

    show_hex("p", p)
    show_hex("g", g)

    a = getrandbits(2048)
    A = pow(g, a, p)
    show_hex("A", A)

    B = input_hex("B")
    show_hex("B", B)
    
    if not (B > 2**1024):
        print("Invalid B value (B <= 2**1024)", file=sys.stderr)
        exit(1)

    s = pow(B, a, p)

    key = s.to_bytes(256, "little")
    assert len(flag) <= len(key)
    ciphertext = strxor(flag, key[:len(flag)])
    show_b64("secret ciphertext", ciphertext)

level6()
```



## level 7

![RSA加密原理](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20240919104732.png)

```python
import base64
from Crypto.Cipher import AES
from Crypto.Hash.SHA256 import SHA256Hash
from Crypto.PublicKey import RSA


ciphertext = "pVGHJ/WtzV58qQ3uY8hFa1PMBBJtv8QxgGQnpZ1cu95biUNjZ4Dg8CrhJ57Qoe26nQ3heKnYUKGZxnA6h3j7DlG0jRjgsD5bOSe5xRo97vWaY6u7M+g9XroWUfmIVhxNsM2pvzR9zIbeHNGMEbdxSVFdxvkRJErAi55jeOT4oRkAIpzlO4OcNm3PGBSwvgsfO5KEfgoupboIh0lAQipNnsLHY4mdLa8Huds/X4IIYwZkmxSJ0r4BPN+xdxLcIC/aoRXdxh0TEGq2WZ+HkXSpzuvItHgwKox3HAW4FWHFpFK+AfLnwuaBxB5lQKduxvgfzRDOl2yCkC1px5Le4rVMQQ=="
ciphertext = base64.b64decode(ciphertext)
ciphertext = int.from_bytes(ciphertext, "little")

d = int.from_bytes(bytes.fromhex("12cd52427217d04ef33577173fa8ef213f48e623b95154bb8a6e23d2a7cdce344c843067cc67875b4e796525b85e26692ba597cbe9727f7b90b6a241ca65558ca9995336f95a3855a1509f4dbedf24353293d1152b1d1e47937a62f212fbaaddc2bf8cf1685e69c95170aacd9cb94941054147c2101fb5f41e419ce2ed160e617d6e31b7d90857af656b963c2a59775e26ed9bfa40c02667009167f119b00c99344b003fdfab547a0deb6613d65b1c0f5f082819dc933d6e67e34cb0ff9007755664cf4b33e7bf8efaaa479e8fec02fd5b74f278890d3db4325cf670ea49946075a1b8a135ffe25ed5e133d866256adde21a65d191aeda63d5d4cb515cd323eb"))
n = int.from_bytes(bytes.fromhex("bee8fa7f5e95bdace37097dafc38316d25aa445cde1234fbd2747f818abcac74a2b6c2b3954827eef04317f0d46d05d2bf6690784e26a0d53b66f133f6af2f9d495ad6c86f93de47f91bd8fca7ce770a345ad0259b8c00e66d95d748fedc0022cee4018b9062216cb410e7e068fb60e320511c1d441c66b7ec1431996485f53d82af3382ea802a028a01f53a49d82b1eb2fa6952c562620da2a12ede4be37cff75dd989d7b1c3caa18766eaa86d659da65dfc02c33f27b40c1c4b98c9b607e1fa4362e40d07b743d4fbb51f6d1ede196969b659600c0bbbc3ce5f165e9de4288e148949b109301b5b648afea9895f75d81221e85787fd11ef16e1e5cd8740cd9"))

flag = pow(ciphertext, d, n).to_bytes(256, "little")
print(flag)

```


## level 8

公私钥 mod的是 `phi_n = (p-1) * (q-1)`，真正加解密的时候mod的是`n = p * q`

```python
import base64
from Crypto.PublicKey import RSA
from Crypto.Util.number import inverse

ciphertext = "WH0ZuAZTHrGo8CGtkNqe7gEJctBrhTrdUgcUto2cmbVv0dAP4oqjxWfw5Tf94k4XtegGgbFBQUDsdPiHbKOdO/kFgHAMPK2xvBeVHzxVlVlCfKqli8VA7svBdyrKLR6OqZch54v0d82O87TY16fQTvcC2Xw+S33ol+VlOHuv00Vg28L+rQ8Qovx3NYI2XuAR/oNtL4vDJToVimKW/fh1NE0v6+kncwGnlz8TNQ88jvt5Hdu8sKrQbf/oD5Rukyb2px05DwmVNXKwGGYdKBug4CaWe4KBRBgLlwxv+0UnDpmp62M65Xq8cL/qaKF5AwMxQqY6eY99PuHfY1UKD1lUOg=="
ciphertext = base64.b64decode(ciphertext)
ciphertext = int.from_bytes(ciphertext, "little")

q = int.from_bytes(bytes.fromhex("eb946a9cb604e6de5b9a2965b2d1ac3c09e38f0dd42c88cc41e77debc93467feb967199fc52453c339bc112bd28d4eeeb78c1e0502b956180ad069dcb2aa9d25fbaf721e9b50392b73333c58a0e26ed93a20425191fd7eaf246ff2c927cba936fb52789f3596332e9f73b214ce31d744a30589a8db2d94eaa7ca306022fb527d"))
p = int.from_bytes(bytes.fromhex("bc8586f2f701fddd16366241c61bf88719eb0d203c03154e270a54ce3d48eaf87993468ebab60776fe3feb21050f420bbf56aabd629515a4ba1af5c9aa3f83d28ee9ea47a35e97758468f92df798bb381d448cdaa5254e2704c3eb38c30007b9f1d0759edb464bb49352170ad1ce25714960825516d773891eb1f454ee8bb497"))
phi_n = (p-1) * (q - 1) 
n = p*q

e = 0x10001
d = inverse(e, phi_n)
flag = pow(ciphertext, d, n).to_bytes(256, 'little')
print(flag)
```


## level 9
SHA-256（Secure Hash Algorithm 256-bit）是一个广泛使用的加密散列函数，属于 SHA-2（Secure Hash Algorithm 2）家族。它生成一个 256 位（32 字节）的散列值，是一种单向散列函数，用于确保数据的完整性和真实性。注意这里是256位的散列值是字节流，你不能直接让字节流与字符串进行对比。要么让字符串编码为字节流，要么让字节流解码为字符串。

```python
a = b'Bd'
b = "Bd"
print(a == b)  # False

```

```python
import hashlib
import os
import base64
from Crypto.Hash import SHA256 as SHA256Hash
def find_collision(secret_hash_prefix):
    while True:
        data = os.urandom(32)
        hash = hashlib.sha256(data).digest()
        if hash[:2] == secret_hash_prefix:
            print(hash)
            return data
secret_sha256_prefix_b64 = "QmQ=" #enter yours here
secret_sha256_prefix = base64.b64decode(secret_sha256_prefix_b64)
print(secret_sha256_prefix)
collision = find_collision(secret_sha256_prefix)
collision_b64 = base64.b64encode(collision).decode('utf-8')
print("Collision (b64):", collision_b64)
```


## level 12

`b'\xde\xad'  <==> b'ab' <==> b'\x61\x62'`

```text
# 关于十六进制转字符串的问题
b'\xde\xad'  <==> b'ab'在形式上是一样的，只不过它通过ASCII码解码了一下，以后见到b'',里面不带\x的字母一律当将其转换成字母的ASCII码表示。
```

像你这种`hex(pow(test2,key.d, key.n)).encode()`，这种就是将原来的字符串按照ASCII编码了一遍。

```python
Python 3.11.9 (main, Apr  2 2024, 08:25:04) [GCC 13.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> "test".encode()
b'test'
>>> bytes.fromhex("test")
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ValueError: non-hexadecimal number found in fromhex() arg at position 0
>>> bytes.fromhex("6162")
b'ab'
>>> 

```


## level 13

python中大小端序的区别

1024 转换为
- 大端序（**最高有效字节存放在内存地址的最低位置**）是 `\x04\x00`
- 小端序（**最低有效字节存放在内存中地址最低的位置**）是 `\x00\x04`

如果加密的过程中密文是以小端序加载的，那么解密的过程中，密文也要以小端序加载。一定要对应上。

## level 14

 In this challenge you will perform a simplified Transport Layer Security (TLS) handshake, acting as the server.    

   You will be provided with Diffie-Hellman parameters, a self-signed root certificate, and the root private key.
    The client will request to establish a secure channel with a particular name, and initiate a Diffie-Hellman key exchange.
    The server must complete the key exchange, and derive an AES-128 key from the exchanged secret.
    Then, using the encrypted channel, the server must supply the requested user certificate, signed by root.
    Finally, using the encrypted channel, the server must sign the handshake to prove ownership of the private user key.

首先是client和server通过Diffie-Hellman 密钥交换，获得共享密钥。然后通过共享密钥生成的AES对称加密的密钥。

之后client向服务器发送证书（用AES对称加密），证书签名（用AES对证书哈希以后的字符串用server的RSA私钥签名），以及用户签名（对Diffie-Hellman 密钥中的A、B以及用户名，使用AES对称加密，再用client的RSA私钥签名）

然后就是server的验证，使用AES解密证书，使用server的公钥验证证书签名，使用user的公钥验证用户签名。

```python

from Crypto.Cipher import AES
from Crypto.Hash.SHA256 import SHA256Hash
from Crypto.PublicKey import RSA
from Crypto.Random import get_random_bytes
import base64
import json
from Crypto.Random import get_random_bytes
from Crypto.Random.random import getrandbits, randrange
from Crypto.Util.Padding import pad, unpad


def show(name, value, *, b64=True):
    print(f"{name}: {value}")


def show_b64(name, value):
    show(f"{name} (b64)", base64.b64encode(value).decode())

def show_hex_block(name, value, byte_block_size=16):
    value_to_show = ""

    for i in range(0, len(value), byte_block_size):
        value_to_show += f"{value[i:i+byte_block_size].hex()}"
        value_to_show += " "
    show(f"{name} (hex)", value_to_show)


def show_hex(name, value):
    show(name, hex(value))

p = int.from_bytes(bytes.fromhex(
    "FFFFFFFF FFFFFFFF C90FDAA2 2168C234 C4C6628B 80DC1CD1 "
    "29024E08 8A67CC74 020BBEA6 3B139B22 514A0879 8E3404DD "
    "EF9519B3 CD3A431B 302B0A6D F25F1437 4FE1356D 6D51C245 "
    "E485B576 625E7EC6 F44C42E9 A637ED6B 0BFF5CB6 F406B7ED "
    "EE386BFB 5A899FA5 AE9F2411 7C4B1FE6 49286651 ECE45B3D "
    "C2007CB8 A163BF05 98DA4836 1C55D39A 69163FA8 FD24CF5F "
    "83655D23 DCA3AD96 1C62F356 208552BB 9ED52907 7096966D "
    "670C354E 4ABC9804 F1746C08 CA18217C 32905E46 2E36CE3B "
    "E39E772C 180E8603 9B2783A2 EC07A28F B5C55DF0 6F4C52C9 "
    "DE2BCBF6 95581718 3995497C EA956AE5 15D22618 98FA0510 "
    "15728E5A 8AACAA68 FFFFFFFF FFFFFFFF"
), "big")

g = 2


b = getrandbits(2048)
B = pow(g, b, p)
show_hex("B", B)
A = int("cf9c1a7ab2a232bc7bd00767d47b80c6a7060a4be6ed1e300d2fc221ecb5f03b99ecace9ca410dd54f3ce2c7bc6c01562bde8fe33802db3f1bf3ec9eea6e5b94e83b026c1b4eaebc6c93e1d7dffb510ca178f13cd8a0282dcde62298f9927e731deccba33846900ce8ba27064bd825d6a8e9866f044d9a133cb25b86c490ba9dd69f98f87b0811e6bae42bbe8ec5b39e4ba70ef587ceac023419199e5d59fef16d32ce0c71ca87955a7be8e576c6ab37967db4085e80eca4d8914dbcf0129ccc0a25fd4027d42b2b52284738867bbcfa381deadcd259f323b7fec341a37b36b6d3bf91b3b8dab3797ecab100905181f75d34e23f96bb1817706c2a2a3e24267b",16)
s = pow(A, b, p)
key = SHA256Hash(s.to_bytes(256, "little")).digest()[:16]
print(f"key : {key}")


cipher_encrypt = AES.new(key=key, mode=AES.MODE_CBC, iv=b"\0"*16)
cipher_decrypt = AES.new(key=key, mode=AES.MODE_CBC, iv=b"\0"*16)
def encrypt_input_b64(data):
    data = base64.b64decode(data)
    try:
        return unpad(cipher_encrypt.encrypt(data), cipher_encrypt.block_size)
    except ValueError as e:
        print(f"{e}", file=sys.stderr)
        exit(1)

import IPython
IPython.embed()

```



```python

from Crypto.Cipher import AES
from Crypto.Hash.SHA256 import SHA256Hash
from Crypto.PublicKey import RSA
from Crypto.Random import get_random_bytes
import base64
import json
from Crypto.Random import get_random_bytes
from Crypto.Random.random import getrandbits, randrange
from Crypto.Util.Padding import pad, unpad

user_key = RSA.generate(1024)
user_certificate = {
    "name": "gjgdlbujheoykcwn",
    "key": {
        "e" : user_key.e,
        "n" : user_key.n
    },
    "signer": "root"
}

root_certificate_b64 = "eyJuYW1lIjogInJvb3QiLCAia2V5IjogeyJlIjogNjU1MzcsICJuIjogMTgyMTgzMjEzMjMwMTU1Mzk2NDIxMzg5MzkwODI4NDY5MTcyMDAzMjc0ODY5NDE3NTcxNzIxOTE2NDgzMzIwNjAyMDUwNDE2NTAxOTM2NDE3NDc5MzAyNjYxODEyOTUxMzMxOTcyMDEzODI2MzEzODY1NDk0MzUyMTU5MTEwNjY2NjQyNTUzMDE3MjcyMzc3ODYzMDA0MTY4NTg5MDc0MzYyODcxMzU2NzI2NzgxODM4MjMzNDE5NzMwMTM5Mjg2MDMxMjEyOTM2OTE1OTU1OTExMDY3OTQ2NTg0OTE5OTU1ODUyOTAzNTk3NTc0MjE4ODg0MTk3ODQ0ODU5MjU0NDA0Nzg0NDg0NDM2NTk3ODMxNjI1MzUyNDQ2MjM3NTg2MTQ0MjU4MDA4MjE4MDQxODE0MDczOTM3ODk0Mjg1ODQwMTk5MDY0MzM0OTQyMDc0ODk3MTU3NDE3NDM5OTIyOTk5MDYzMTIxNDYzMjM3NDUwMTgxMzM3MjkzOTA1NjQ1NjkzNjAyODQ4MDEwNTE0NjE0NzA4MDU4ODU3MDIwNDAxMDM2NDI3NTAyMTYyNzY5NzEyMDQ4MjYyODk3NjA5Nzk1NDgwNjA1NjgyMDQ0NDc4MDMwNzc1OTE3ODk4MDczNTY4MDIwNDQ4MTY5MjA3MTgxODM2NjYxNjM5Njk3NjgyMzI4ODk1Mzg5MzAwNTkyODgwNjU5MDY4ODA0Nzk1MjA1MjE1MjA4NjUwNTgxMzY2MDU5Nzk0NzcxMTgzOTQyNDkxMzUwNDMwNjA1MzcwOTI0ODUyOTQ4Mzc3ODE4MzIxMDc1OTA0NDN9LCAic2lnbmVyIjogInJvb3QifQ=="
root_certificate = json.loads(base64.b64decode(root_certificate_b64))
print(root_certificate)

root_certificate["key"]["d"] = int("34cbf94a5f0a864b565e660227f48d2e0981b477d1ad4a8aca1f7400e0f528fd1499d23fe34091e6047bd24a876e37083fb7853dec1e20fd5cb743068e6ca520dc791efeba3b23f5dad1ef279d74aa408e148933be82e5f9ff1d5a53fbc6991882794989d481a4ce64378e52d78ac3d30bf37976aebed1f65e9dea0b46d98afd0cd27a32df9942a1b498babdc8edfcfa825431b465236d72c919f9db850c9a8c3d2edaaa3174012933c229adaf2557502eb3cdd9335faf4ce2b37e08ae5f0391dafbed7346a58461d2db1c0e56414942fef0de3c37a5a9a1050f604df8528ba3724d91a776009380c1e705ab06108d7903a681e30f6a1d3065fd0ecb8d9a8141", 16)


A = int("cf9c1a7ab2a232bc7bd00767d47b80c6a7060a4be6ed1e300d2fc221ecb5f03b99ecace9ca410dd54f3ce2c7bc6c01562bde8fe33802db3f1bf3ec9eea6e5b94e83b026c1b4eaebc6c93e1d7dffb510ca178f13cd8a0282dcde62298f9927e731deccba33846900ce8ba27064bd825d6a8e9866f044d9a133cb25b86c490ba9dd69f98f87b0811e6bae42bbe8ec5b39e4ba70ef587ceac023419199e5d59fef16d32ce0c71ca87955a7be8e576c6ab37967db4085e80eca4d8914dbcf0129ccc0a25fd4027d42b2b52284738867bbcfa381deadcd259f323b7fec341a37b36b6d3bf91b3b8dab3797ecab100905181f75d34e23f96bb1817706c2a2a3e24267b", 16)
B = int("3a495db7bc61c676709f7f661a4ea9d50fcd5240b40231be53353cf8d4ab454158b286a4b5c828be6a13a82078d3c15fb7ff060a54ca26c38aac81d20040fd7aa44d84f2fddcff1da2a8dffc2a9c460506cd6e19915e6dc4d99ce6f2d84016b79f2221f53d7eca442ba9e6e360e5bdf316ace4bbc8305fceeaa9aa0fd12c8371e480e7bfd7ed2be3c5de1d7fa2e8d23bbc12598a2c9b6d4d20b93762b74fe1a1fbf936f57b519ecda128f54c4aa496a02e3f5f71c2290c1f5d1e5c0fd69172d3d8f67f849a5f6e44384f958200e2e54d3156c73db4cf354a85e0badd6a4eedfb9a4a11d4412dd389392e0dc4538a67b59584ca2e83650aee6d1df91f8f1e5d80", 16)
user_signature_data = (
    "gjgdlbujheoykcwn".encode().ljust(256, b"\0") +
    A.to_bytes(256, "little") +
    B.to_bytes(256, "little")
)
user_signature_hash = SHA256Hash(user_signature_data).digest()
user_signature = pow(
    int.from_bytes(user_signature_hash, "little"),
    user_key.d,
    user_key.n
).to_bytes(256, "little")


key = b'\x06\x87G=\xb4<\x85\x9d&\xd5\x17.\xb4\xf4\xb6o'
print(f"key : {key}")
cipher_encrypt = AES.new(key=key, mode=AES.MODE_CBC, iv=b"\0"*16)
cipher_decrypt = AES.new(key=key, mode=AES.MODE_CBC, iv=b"\0"*16)
def encrypt_input_b64(data):
    data = cipher_encrypt.encrypt(pad(data, cipher_encrypt.block_size))
    try:
        return base64.b64encode(data)
    except ValueError as e:
        print(f"{e}", file=sys.stderr)
        exit(1)

user_certificate_hash = SHA256Hash(json.dumps(user_certificate).encode()).digest()
user_certificate_signature = pow(
    int.from_bytes(user_certificate_hash, "little"),
    root_certificate["key"]["d"],
    root_certificate["key"]["n"]
).to_bytes(256, "little")

print(json.dumps(user_certificate).encode())
user_certificate_data = encrypt_input_b64(json.dumps(user_certificate).encode())
user_certificate_signature = encrypt_input_b64(user_certificate_signature)
user_signature = encrypt_input_b64(user_signature)

print(f"user_certificate_data:{user_certificate_data}")
print(f"user_certificate_signature:{user_certificate_signature}")
print(f"user_signature:{user_signature}")


import IPython
IPython.embed()



```



