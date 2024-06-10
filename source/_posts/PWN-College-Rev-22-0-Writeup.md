---
title: PWN-College-Writeup
date: 2024-06-10 22:34:00
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

https://pwn.college/program-security/reverse-engineering

![20240610223520](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20240610223520.png)


## 遗留思考的问题

1. Linux的权限管理是如何管理的，用户权限、读写权限、运行程序的权限、以及程序是否拥有读取文件的权限？

2. 调试器和被调试程序之间的关系？如何完成调试过程，他们之前的权限关系是怎么样的？

## 尝试

由于程序在最初会读取`flag`文件的内容去生成种子，所以我想能不能通过调试这个程序来获取其在内存中存放的`flag`文件的内容。但还是权限的问题，因为本身我`hacker`用户是没有读取`flag`文件的权限的，而`baby`这个程序拥有读取`flag`文件的权限，我拥有运行`baby`程序的权限。但是调试程序虽然可以调试`baby`程序，可是调试程序没有读取`flag`文件的权限，所以调试程序哪怕可以调试`baby`程序，依旧会在程序断言处报错。

```c++
  fd = open("/flag", 0);
  if ( fd < 0 )
    __assert_fail("fd >= 0", "<stdin>", 0x11u, "flag_seed");
  if ( read(fd, buf, 0x80uLL) <= 0 )
    __assert_fail("read(fd, flag, 128) > 0", "<stdin>", 0x12u, "flag_seed");
```

## 解题思路

这个跟以往不同的是，各个寄存器、系统调用、指令类别对应的特征数之前是确定的。比如`a`对应`0x01`、`imm`对应`0x01`。现在它会根据flag文件中的内容生成一个数，然后将这个数作为种子，随机确定各个寄存器、系统调用、指令类别所对应的特征数。

观察下面这个函数，就可以发现这个`seed`在flag文件中的内容确定之后，就变成确定的了。而根据之前对随机函数的了解，如果种子是确定的话，那么多次运行生成的随机数序列是相同的，譬如无论运行多少次，生成的都是`1、2、3、4`。

所以，后续我们要做的，就是根据特征数`0x1、0x2、0x4、0x8、0x10、0x20、0x40、0x80`尝试其对应的寄存器、系统调用、指令类别。

```c++
unsigned __int64 flag_seed()
{
  unsigned int seed; // [rsp+4h] [rbp-9Ch]
  unsigned int i; // [rsp+8h] [rbp-98h]
  int fd; // [rsp+Ch] [rbp-94h]
  __int64 buf[17]; // [rsp+10h] [rbp-90h] BYREF
  unsigned __int64 v5; // [rsp+98h] [rbp-8h]

  v5 = __readfsqword(0x28u);
  memset(buf, 0, 128);
  fd = open("/flag", 0);
  if ( fd < 0 )
    __assert_fail("fd >= 0", "<stdin>", 0x11u, "flag_seed");
  if ( read(fd, buf, 0x80uLL) <= 0 )
    __assert_fail("read(fd, flag, 128) > 0", "<stdin>", 0x12u, "flag_seed");
  seed = 0;
  for ( i = 0; i <= 0x1F; ++i )
    seed ^= *((_DWORD *)buf + (int)i);
  srand(seed);
  memset(buf, 0, 0x80uLL);
  return __readfsqword(0x28u) ^ v5;
}
```


没想到这次居然一次就过了，看来还是有在慢慢进步的！

```c
def write_binary_file(file_path, data_list):
    # 以二进制写模式打开文件
    with open(file_path, 'wb') as f:
        for data in data_list:
            # 将二进制数据写入文件
            f.write(data)

# 示例数据：将几个二进制块写入文件
data_list = [
    # b'\x20\x20\x20' # STM
    b'\x40\x10\x10' # IMM a = \x10
    b'\x40\x2f\x08' # IMM b = \x2f
    b'\x01\x08\x10' # stm *a = b

    b'\x40\x11\x10' # IMM a = \x11
    b'\x40\x66\x08' # IMM b = \x66
    b'\x01\x08\x10' # stm *a = b

    b'\x40\x12\x10' # IMM a = \x12
    b'\x40\x6c\x08' # IMM b = \x6c
    b'\x01\x08\x10' # stm *a = b
    
    b'\x40\x13\x10' # IMM a = \x13
    b'\x40\x61\x08' # IMM b = \x61
    b'\x01\x08\x10' # stm *a = b

    b'\x40\x14\x10' # IMM a = \x14
    b'\x40\x67\x08' # IMM b = \x67
    b'\x01\x08\x10' # stm *a = b
    
    b'\x40\x15\x10' # IMM a = \x15
    b'\x40\x00\x08' # IMM b = \x00
    b'\x01\x08\x10' # stm *a = b

    b'\x40\x00\x08' # IMM b = \x00
    b'\x40\x10\x10' # IMM a = \x10
    b'\x20\x10\x80' # open  -- return file -> a

    b'\x40\x60\x40', # imm c = 0x60  
    b'\x40\x20\x08', # imm b = 0x20  
    b'\x20\x10\x10', # read(a, b, c) --return bytes -> a


    b'\x40\x60\x40', # imm c = 0x60   
    b'\x40\x01\x10', # imm a = 0x01 
    b'\x20\x10\x02', # write  ---return bytes -> a

    b'\x20\x10\x20' # exit
]

"""
STM \x01       i    sleep
JMP \x02       f    write    
STK \x04       d     
CMP \x08       b    read_code
LDM \x10       a    read_memory
SYS \x20            exit
IMM \x40       c
ADD \x80       s    open

"""
# 目标文件路径
file_path = r'/home/hacker/Test/output3'

# 调用函数写入二进制数据
write_binary_file(file_path, data_list)

# 验证写入的数据
with open(file_path, 'rb') as f:
    content = f.read()
    print(content)

```