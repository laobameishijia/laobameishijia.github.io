---
title: PWN-College-Web-4-Writeup
date: 2024-09-12 10:46:00
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

https://pwn.college/intro-to-cybersecurity/access-control/

## 有价值的问题

### 1. 文件权限

https://www.runoob.com/linux/linux-file-attr-permission.html很详细的讲解

```bash
os.chmod("/bin/cat", 0o4755)
After:
-rwsr-xr-x 1 root root 43416 Sep  5  2019 /bin/cat
```



**`4` (Set-UID 位)**：这是设置用户 ID（Set-UID）位。当一个可执行文件设置了 Set-UID 位时，无论哪个用户运行该文件，都会以文件所有者（通常是 `root`）的身份运行它。

**`2` (Set-GID 位)**：这是设置组 ID（Set-GID）位。设置 Set-GID 位的文件在执行时，会以文件所属组的权限执行，用户仍然以自己的用户身份运行，但会获得文件所属组的权限。

​	

## level 5

cp 复制文件中有一个选项可以不保持原有文件的劝降

```bash
hacker@access-control~level5:~$ cp --no-preserve=mode /flag ./fla
hacker@access-control~level5:~$ ls -l ./fla 
-rw-r--r-- 1 root hacker 58 Sep 12 03:26 ./fla
hacker@access-control~level5:~$ cat ./fla
pwn.college{A_d8ZpmLAnm6Z3lcKxbTkBx9jWn.dZjM4MDL0czNxEzW}
hacker@access-control~level5:~$ ls -l /flag
-r-------- 1 root root 58 Sep 12 03:18 /flag
hacker@access-control~level5:~$ 
```

## level 6

```python
acker@access-control~level6:~$ /challenge/run 
===== Welcome to Access Control! =====
In this series of challenges, you will be working with various access control systems.
Break the system to get the flag.


In this challenge you will work with different UNIX permissions on the flag.

The flag file is owned by root and a new group.

Hint: Search for how to join a group with a password.


Before:
-r-------- 1 root root 58 Sep 12 03:38 /flag
After:
----r----- 1 root group_qnqnmafv 58 Sep 12 03:38 /flag
The password for group_qnqnmafv is: fjochprb
hacker@access-control~level6:~$ id
uid=1000(hacker) gid=1000(hacker) groups=1000(hacker)
hacker@access-control~level6:~$ newgrp group_qnqnmafv
Password: 
Note: Your home directory is running low on storage:
Filesystem                             Size  Used Avail Use% Mounted on
192.168.42.1:/data/homes/mounts/11774  982M  600M  316M  66% /home/hacker

Filling your home directory completely could cause you to lose access to the workspace and/or desktop.
You can view a list of the largest files and directories using the command:
  du -sh /home/hacker/{*,.*} | sort -h
hacker@access-control~level6:~$ id
uid=1000(hacker) gid=1001(group_qnqnmafv) groups=1001(group_qnqnmafv),1000(hacker)
```



## level 17

编程的过程中出现了一个问题:  -----这个地方`{''}`并不是空集
```python
hacker@access-control~level17:~$ python
Python 3.11.9 (main, Apr  2 2024, 08:25:04) [GCC 13.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> a = {"a"}
>>> b = {''}
>>> print(a >= b)
False
>>> a = {"a", "b"}
>>> b = {'b'}
>>> print(a >= b)
True
>>> 
```


学习一下，源代码的编程思路。
```python
def is_subset_equal(a: int, b: int) -> bool:
    return (b | a) == b # 来判断 a 是不是 b 的子集。  a/b中的每一位都代表一种类别设置。

LEVELS = [Level("TS", 4),
        Level("S", 3),
        Level("C", 2),
        Level("UC", 1)]

CATEGORIES = [
        Category("NUC", 1),
        Category("NATO", 1 << 1),
        Category("ACE", 1 << 2),
        Category("UFO", 1 << 3)
    ]

is_allowed_by_level = subject.level >= object.level if is_read else object.level >= subject.level
is_allowed = False

if is_allowed_by_level:
    if is_read:
        is_allowed = is_subset_equal(object_set, subject_set)
    else:
        is_allowed = is_subset_equal(subject_set, object_set)

if is_allowed == answer_allowed:
    print("Correct!")
else:
    print("Incorrect!")
    exit(-1)


```

## Level 19

```python
from pwn import *
import re

class Level:
    def __init__(self, name: str, level: int):
        self.name = name
        self.level = level

class Category:
    def __init__(self, name: str, bit: int):
        self.name = name
        self.bit = bit

def is_subset_equal(a: int, b: int) -> bool:
    return (b | a) == b

def judge_correct(user_1, user_2, is_read):
    subject = LEVELS[user_1[0]]
    object =  LEVELS[user_2[0]]

    subject_set = user_1[1]
    object_set = user_2[1]

    is_allowed_by_level = subject.level >= object.level if is_read else object.level >= subject.level
    is_allowed = False

    if is_allowed_by_level:
        if is_read:
            is_allowed = is_subset_equal(object_set, subject_set)
        else:
            is_allowed = is_subset_equal(subject_set, object_set)
    
    return is_allowed


LEVELS = {}
CATEGORIES = {}

p = process("/challenge/run")
p.recvuntil(b"(first is highest aka more sensitive):\n")
for i in range(40):
    name = p.recvline().decode("utf-8").strip()
    print(f"Level {name}")
    level= Level(name, 40 - i)
    LEVELS[f"{name}"] = level

p.recvuntil(b"5 Categories:\n")
for i in range(5):
    name = p.recvline().decode("utf-8").strip()
    print(f"Category {name}")
    category = Category(name, 1 << (5 -i))
    CATEGORIES[f"{name}"] = category

# print(LEVELS)
# print(CATEGORIES)

pattern = r'level (\w+) and categories \{([^}]*)\}'
p.recvuntil("Q ")
for i in range(128):
    user_1 = []
    user_2 = []
    question = p.recvline().decode("utf-8")
    print(f"{i} : {question}")
    # 查找所有匹配项
    matches = re.findall(pattern, question)
    for j, match in enumerate(matches):
        level = match[0]
        categories_set = set(match[1].split(", "))
        set_ = 0
        for categories in categories_set:
            if categories == '':
                continue
            set_ |= CATEGORIES[categories].bit
        if j == 0:
            user_1.append(level)
            user_1.append(set_)
        else:
            user_2.append(level)
            user_2.append(set_)
        
    if judge_correct(user_1, user_2, "read" in question):
        p.sendline("yes")
        print("yes")
    else:
        p.sendline("no")
        print("no")
    if i == 127:
        p.interactive()
    p.recvuntil("Q ")


```