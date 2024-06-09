---
title: PWN-College-Writeup
date: 2024-06-08 17:49:00
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


![20240608175412](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20240608175412.png)


这里面有个地方需要注意： 

这个里面`a2`的值是我可以控制的，如果是这样的话，岂不是任意地址都可以写入了？这样做会不会破坏程序的运行空间？
```c
__int64 __fastcall write_memory(__int64 a1, unsigned __int8 a2, char a3)
{
  __int64 result; // rax

  result = a2;
  *(_BYTE *)(a1 + a2 + 0x300) = a3;
  return result;
}
```

用户输入（模拟架构程序）是放在`buf`变量里面的，`buf`变量放在栈空间上。它的大小有`0x400`，那模拟程序架构的代码空间大小为`0x300`，`0x300`之后的空间是模拟架构寄存器的存放空间以及模拟内存。

后面代码写入`\flag`的位置，即是在`0x300+0x10`的位置。

## 错误

一开始我没打算往内存中写入`\flag`，想利用模拟程序的栈空间写入`\flag`。

后来我发现open函数的第一个参数是应该是`\flag`首字符的地址，但是栈空间的地址我没有办法知道。

```c++
  if ( (a2 & 0x80000) != 0 )
  {
    puts("[s] ... open");
    v3 = open((const char *)&a1[a1[1024] + 768], a1[1025], a1[1026]);
    write_register(a1, (unsigned __int8)a2, v3);
  }
```

后来，发现，只能利用模拟程序的写入内存操作，将字符串写入特定的地址，然后再把地址传递给open函数

```c++

  LOBYTE(v2) = read_register(a1, (unsigned __int8)a2);
  v4 = read_register(a1, BYTE2(a2));
  return write_memory(a1, v4, (char)v2);

// write_memory
__int64 __fastcall write_memory(__int64 a1, unsigned __int8 a2, char a3)
{
  __int64 result; // rax

  result = a2;
  *(_BYTE *)(a1 + a2 + 0x300) = a3;
  return result;
}  

```

## 21

本关卡程序需要自己来写，之前关卡的程序思路是接受你的输入，然后通过算法判断输入与内存数据是否相等，相等则打开flag文件显示flag。

flag文件所在的位置在根目录，hacker用户没有访问权限，但是关卡所给出的二进制程序可以访问。

所以我们的思路也很明确，利用模拟架构向程序写入 `/flag`即flag文件的路径，然后通过模拟架构的系统调用，打开文件、读取文件、向标准输出显示。

```python
def write_binary_file(file_path, data_list):
    # 以二进制写模式打开文件
    with open(file_path, 'wb') as f:
        for data in data_list:
            # 将二进制数据写入文件
            f.write(data)

# 示例数据：将几个二进制块写入文件
data_list = [
    b'\x10\x40\x10',# imm a = 0x10
    b'\x2f\x40\x08',# imm b = 0x2f
    b'\x08\x80\x10',# stm *a = b
  
    b'\x11\x40\x10',# imm a = 0x11
    b'\x66\x40\x08',# imm b = 0x66
    b'\x08\x80\x10',# stm *a = b


    b'\x12\x40\x10',# imm a = 0x12
    b'\x6c\x40\x08',# imm b = 0x6c
    b'\x08\x80\x10',# stm *a = b
  
  
    b'\x13\x40\x10',# imm a = 0x13
    b'\x61\x40\x08',# imm b = 0x61
    b'\x08\x80\x10',# stm *a = b
  
    b'\x14\x40\x10',# imm a = 0x14
    b'\x67\x40\x08',# imm b = 0x67
    b'\x08\x80\x10',# stm *a = b
  
    b'\x15\x40\x10',# imm a = 0x15
    b'\x00\x40\x08',# imm b = 0x00
    b'\x08\x80\x10',# stm *a = b

  
    b'\x10\x40\x08', # imm b = 0x00
    b'\x10\x40\x10', # imm a = 0x10      
    b'\x10\x01\x08', # open
   
    b'\x60\x40\x01', # imm c = 0x60  
    b'\x20\x40\x08', # imm b = 0x20  
    b'\x10\x01\x20', # read(a, b, c)
  
    b'\x60\x40\x01', # imm c = 0x60   
    b'\x00\x40\x10', # imm a = 0x01 标准输出去写
    b'\x01\x01\x02', #write
  
    b'\x01\x01\x01', #exit

]

# 目标文件路径
file_path = r'.\output.bin'

# 调用函数写入二进制数据
write_binary_file(file_path, data_list)

# 验证写入的数据
with open(file_path, 'rb') as f:
    content = f.read()
    print(content)


"""
[+] Welcome to challenge/babyrev_level21.0!
[+] This challenge is an custom emulator. It emulates a completely custom
[+] architecture that we call "Yan85"! You'll have to understand the
[+] emulator to understand the architecture, and you'll have to understand
[+] the architecture to understand the code being emulated, and you will
[+] have to understand that code to get the flag. Good luck!
[+]
[+] This level is a full Yan85 emulator. You'll have to reason about yancode,
[+] and the implications of how the emulator interprets it!
[!] This time, YOU'RE in control! Please input your yancode: [+] Starting interpreter loop! Good luck!
[V] a:0 b:0 c:0 d:0 s:0 i:0x1 f:0
[I] op:0x40 arg1:0x10 arg2:0x10
[s] IMM a = 0x10
[V] a:0x10 b:0 c:0 d:0 s:0 i:0x2 f:0
[I] op:0x40 arg1:0x8 arg2:0x2f
[s] IMM b = 0x2f
[V] a:0x10 b:0x2f c:0 d:0 s:0 i:0x3 f:0
[I] op:0x80 arg1:0x10 arg2:0x8
[s] STM *a = b
[V] a:0x10 b:0x2f c:0 d:0 s:0 i:0x4 f:0
[I] op:0x40 arg1:0x10 arg2:0x11
[s] IMM a = 0x11
[V] a:0x11 b:0x2f c:0 d:0 s:0 i:0x5 f:0
[I] op:0x40 arg1:0x8 arg2:0x66
[s] IMM b = 0x66
[V] a:0x11 b:0x66 c:0 d:0 s:0 i:0x6 f:0
[I] op:0x80 arg1:0x10 arg2:0x8
[s] STM *a = b
[V] a:0x11 b:0x66 c:0 d:0 s:0 i:0x7 f:0
[I] op:0x40 arg1:0x10 arg2:0x12
[s] IMM a = 0x12
[V] a:0x12 b:0x66 c:0 d:0 s:0 i:0x8 f:0
[I] op:0x40 arg1:0x8 arg2:0x6c
[s] IMM b = 0x6c
[V] a:0x12 b:0x6c c:0 d:0 s:0 i:0x9 f:0
[I] op:0x80 arg1:0x10 arg2:0x8
[s] STM *a = b
[V] a:0x12 b:0x6c c:0 d:0 s:0 i:0xa f:0
[I] op:0x40 arg1:0x10 arg2:0x13
[s] IMM a = 0x13
[V] a:0x13 b:0x6c c:0 d:0 s:0 i:0xb f:0
[I] op:0x40 arg1:0x8 arg2:0x61
[s] IMM b = 0x61
[V] a:0x13 b:0x61 c:0 d:0 s:0 i:0xc f:0
[I] op:0x80 arg1:0x10 arg2:0x8
[s] STM *a = b
[V] a:0x13 b:0x61 c:0 d:0 s:0 i:0xd f:0
[I] op:0x40 arg1:0x10 arg2:0x14
[s] IMM a = 0x14
[V] a:0x14 b:0x61 c:0 d:0 s:0 i:0xe f:0
[I] op:0x40 arg1:0x8 arg2:0x67
[s] IMM b = 0x67
[V] a:0x14 b:0x67 c:0 d:0 s:0 i:0xf f:0
[I] op:0x80 arg1:0x10 arg2:0x8
[s] STM *a = b
[V] a:0x14 b:0x67 c:0 d:0 s:0 i:0x10 f:0
[I] op:0x40 arg1:0x10 arg2:0x15
[s] IMM a = 0x15
[V] a:0x15 b:0x67 c:0 d:0 s:0 i:0x11 f:0
[I] op:0x40 arg1:0x8 arg2:0
[s] IMM b = 0
[V] a:0x15 b:0 c:0 d:0 s:0 i:0x12 f:0
[I] op:0x80 arg1:0x10 arg2:0x8
[s] STM *a = b
[V] a:0x15 b:0 c:0 d:0 s:0 i:0x13 f:0
[I] op:0x40 arg1:0x8 arg2:0x10
[s] IMM b = 0x10
[V] a:0x15 b:0x10 c:0 d:0 s:0 i:0x14 f:0
[I] op:0x40 arg1:0x10 arg2:0x10
[s] IMM a = 0x10
[V] a:0x10 b:0x10 c:0 d:0 s:0 i:0x15 f:0
[I] op:0x1 arg1:0x8 arg2:0x10
[s] SYS 0x8 a
[s] ... open
[s] ... return value (in register a): 0x3
[V] a:0x3 b:0x10 c:0 d:0 s:0 i:0x16 f:0
[I] op:0x40 arg1:0x1 arg2:0x60
[s] IMM c = 0x60
[V] a:0x3 b:0x10 c:0x60 d:0 s:0 i:0x17 f:0
[I] op:0x40 arg1:0x8 arg2:0x20
[s] IMM b = 0x20
[V] a:0x3 b:0x20 c:0x60 d:0 s:0 i:0x18 f:0
[I] op:0x1 arg1:0x20 arg2:0x10
[s] SYS 0x20 a
[s] ... read_memory
[s] ... return value (in register a): 0x39
[V] a:0x39 b:0x20 c:0x60 d:0 s:0 i:0x19 f:0
[I] op:0x40 arg1:0x1 arg2:0x60
[s] IMM c = 0x60
[V] a:0x39 b:0x20 c:0x60 d:0 s:0 i:0x1a f:0
[I] op:0x40 arg1:0x10 arg2:0x1
[s] IMM a = 0x1
[V] a:0x1 b:0x20 c:0x60 d:0 s:0 i:0x1b f:0
[I] op:0x1 arg1:0x2 arg2:0x1
[s] SYS 0x2 c
[s] ... write
pwn.college{U12e5jno_xl8vQmh9fN-p_fqO_Q.0VN4IDL0czNxEzW}
[s] ... return value (in register c): 0x60
[V] a:0x1 b:0x20 c:0x60 d:0 s:0 i:0x1c f:0
[I] op:0x1 arg1:0x1 arg2:0x1
[s] SYS 0x1 c
[s] ... exit


"""
```

逻辑还是比较简单的，是一个位移加密的操作.

```python



adder = [
    0x39,0xf8,0xe1,0x75,
    0x32,0x91,0xe8,0x22,
    0x22,0x8a,0xdc,0xaa,
    0x83,0xb7,0x67,0x83,
    0xad,0x8d,0xf6,0x06,
    0xba
]

def read_binary_file_in_chunks(file_path, start_position=0x39a ,chunk_size=3):
    with open(file_path, 'rb') as f:  # 以二进制模式打开文件
        while True:
            f.seek(start_position)  # 定位到指定的开始位置
            chunk = f.read(chunk_size)  # 每次读取三个字节
            if not chunk:
                break  # 到达文件末尾，退出循环
            return chunk  # 使用生成器返回读取的字节块

filepath = r"F:\研二上\coding\ctf\pwncollege\command"

res = [None] * 21 

num = 0
while(True):
  
    byte = read_binary_file_in_chunks(file_path=filepath, start_position=0x386+num, chunk_size=1)
    if num == 21 : 
        break
    res[num] = (ord(byte) - adder[num]) % 256
  
    if res[num] < 0 : 
        res[num] += 0xff
  
    res[num] = hex(res[num])
  
    num += 1

print(res)
```

![20240606171440](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20240606171440.png)

## 21.2

第二关跟上面的一样，

指令的组成形式由`arg_0 op arg_1`变成了`op arg_0 arg_1`，所以二进制指令的需要重新调整一下，另外模拟函数调用的字节特征也发生了变化，也需要进行相应的调整。

```c++
def write_binary_file(file_path, data_list):
    # 以二进制写模式打开文件
    with open(file_path, 'wb') as f:
        for data in data_list:
            # 将二进制数据写入文件
            f.write(data)

# 示例数据：将几个二进制块写入文件
data_list = [

    b'\x08\x10\x08',# imm a = 0x10
    b'\x08\x2f\x02',# imm b = 0x2f
    b'\x10\x02\x08',# stm *a = b

    b'\x08\x11\x08',# imm a = 0x11
    b'\x08\x66\x02',# imm b = 0x66
    b'\x10\x02\x08',# stm *a = b


    b'\x08\x12\x08',# imm a = 0x12
    b'\x08\x6c\x02',# imm b = 0x6c
    b'\x10\x02\x08',# stm *a = b
    
    
    b'\x08\x13\x08',# imm a = 0x13
    b'\x08\x61\x02',# imm b = 0x61
    b'\x10\x02\x08',# stm *a = b
    
    b'\x08\x14\x08',# imm a = 0x14
    b'\x08\x67\x02',# imm b = 0x67
    b'\x10\x02\x08',# stm *a = b
    
    b'\x08\x15\x08',# imm a = 0x15
    b'\x08\x00\x02',# imm b = 0x00
    b'\x10\x02\x08',# stm *a = b

  
    b'\x08\x10\x08', # imm b = 0x00
    b'\x08\x10\x02', # imm a = 0x10        
    b'\x40\x08\x20', # open
   
    b'\x08\x60\x10', # imm c = 0x60  
    b'\x08\x20\x02', # imm b = 0x20  
    b'\x40\x08\x01', # read(a, b, c)
    
    b'\x08\x60\x10', # imm c = 0x60   
    b'\x08\x01\x08', # imm a = 0x01 
    b'\x40\x08\x08', # write
    
    b'\x40\x01\x10', #exit

]


# 目标文件路径
file_path = r'/home/hacker/Test/output2'

# 调用函数写入二进制数据
write_binary_file(file_path, data_list)

# 验证写入的数据
with open(file_path, 'rb') as f:
    content = f.read()
    print(content)

```

![20240609214816](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20240609214816.png)