---
title: PWN-College-Web-4-Writeup
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
```
![20240917122217](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20240917122217.png)


### 2. pwntool的用法

`context.log_level = debug`，这个可以显示与进程交互的所有信息，全部输入输出的回显。



### 3. Base64 编码原理

https://juejin.cn/post/6994612829437296647


### 4. AES加密


AES（高级加密标准，Advanced Encryption Standard） 是一种对称加密算法。在对称加密中，加密和解密使用相同的密钥。发送方使用密钥将明文加密为密文，接收方则使用相同的密钥将密文解密回明文。因为密钥必须在加密和解密双方之间安全共享，所以对称加密要求双方拥有相同的密钥。

![20240916172338](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20240916172338.png)

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



## 
