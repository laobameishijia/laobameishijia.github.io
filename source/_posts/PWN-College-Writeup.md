---
title: PWN-College-Writeup
date: 2024-06-06 17:03:00
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

已经很久没有动笔写过东西了，马上要找工作了，把之前的东西捡一捡。

https://pwn.college/program-security/reverse-engineering

## 20.1 

从这关开始，难度就大了一些，因为模拟的程序架构没有提示回显了。需要自行分析。这也是我第一次完整的把这个自定义架构理解明白，并且通过python复现了这个架构，复现的过程中，对于程序指令又有了一些新的体会，简单的IMM、ADD、STM、STK在加上一些系统调用居然可以自己定义。把自己的代码通过二进制的方式放在程序中，然后在运行的过程中，使用自己的定义的架构指令进行解析运行。

下面是解析文件中存放二进制程序指令内容的程序，期间将各个寄存器、栈指针、以及相应的提示进行显示。

```python
def read_binary_file_in_chunks(file_path, start_position ,chunk_size=3):
    with open(file_path, 'rb') as f:  # 以二进制模式打开文件
        while True:
            f.seek(start_position*3)  # 定位到指定的开始位置
            chunk = f.read(chunk_size)  # 每次读取三个字节
            if not chunk:
                break  # 到达文件末尾，退出循环
            return chunk  # 使用生成器返回读取的字节块


def get_memory(arg):
    return stack[arg]

def get_char_register(arg):
    if arg == 32: 
        return "a"
    elif arg == 2:
        return "b"
    elif arg == 16:
        return "c"
    elif arg == 8:
        return "d"
    elif arg == 1:
        return "s"
    elif arg == 4:
        return "i"
    else:
        print("unknow register")
        return None

def read_register(arg):
    global a, b ,c ,d, s, i ,f,stack
    if arg == 32: 
        return a
    elif arg == 2:
        return b
    elif arg == 16:
        return c
    elif arg == 8:
        return d
    elif arg == 1:
        return s
    elif arg == 4:
        return i
    else:
        print("unknow register")
        return None

def writer_register(arg0, arg1):
    global a, b ,c ,d, s, i ,f,stack
    if arg0 == 32:
        a = arg1 & 0xff
    elif arg0 == 2:
        b = arg1 & 0xff
    elif arg0 == 16:
        c = arg1 & 0xff
    elif arg0 == 8:
        d = arg1 & 0xff
    elif arg0 == 1:
        s = arg1 & 0xff
    elif arg0 == 4:
        i = arg1 & 0xff
    elif arg0 == 64:
        f = arg1 & 0xff
    else:
        print("unknown register")

def write_to_memeory(arg0, arg1):
    stack[arg0] = arg1
    
def print_register(chunk):
    global a, b ,c ,d, s, i ,f,stack
    arg1, op, arg2 = chunk
    print(f"a:{a:02x} , b:{b:02x}, c:{c:02x}, d:{d:02x}, s:{s:02x}, i:{i:02x}, f:{f:02x}")
    print(f"op:{op:02x} , arg1:{arg1:02x}, arg2:{arg2:02x}")

def imm(chunk):
    global a, b ,c ,d, s, i ,f,stack
    arg1, op, arg2 = chunk
    writer_register(arg1, arg2)
    print(f" IMM {get_char_register(arg1)} {arg2:02x}")
    print_register(chunk)

def add(chunk):
    global a, b ,c ,d, s, i ,f,stack
    arg1, op, arg2 = chunk
    v2 = read_register(arg1)
    v3 = read_register(arg2)
    writer_register(arg1,v2+v3)
    print(f"ADD {get_char_register(arg1)} {get_char_register(arg2)}")
    print_register(chunk)

def stk(chunk):
    global a, b ,c ,d, s, i ,f,stack
    arg1, op, arg2 = chunk
    global s
    if arg2: # push
        s += 1
        v2 = read_register(arg2)
        stack[s] = v2
        print(f"PUSH {get_char_register(arg2)}")
    if arg1: # pop
        v4 = stack[s]
        writer_register(arg1, v4)
        s -= 1
        print(f"POP {get_char_register(arg1)}")
    print_register(chunk)

def stm(chunk):
    global a, b ,c ,d, s, i ,f,stack
    arg1, op, arg2 = chunk
    v2 = read_register(arg2)
    v3 = read_register(arg1)
    write_to_memeory(v3, v2)
    print(f"STM *{get_char_register(arg1)} = {get_char_register(arg2)}")
    print_register(chunk)

def ldm(chunk):
    global a, b ,c ,d, s, i ,f,stack
    arg1, op, arg2 = chunk
    v2 = read_register(arg2)
    v3 = get_memory(v2)
    writer_register(arg1, v3)
    print(f"LDM {get_char_register(arg1)} = *{get_char_register(arg2)}")
    print_register(chunk)

def cmp(chunk):
    global a, b ,c ,d, s, i ,f,stack
    arg1, op, arg2 = chunk
    v3 = read_register(arg1)
    v4 = read_register(arg2)
    f = 0

    if v3 < v4:
        f |= 1
    if v3 > v4:
        f |= 0x10
    if v3 == v4:
        f |= 2
    if v3 != v4:
        f |= 4
    if not v3 and not v4:
        f |= 8
    print(f"CMP {get_char_register(arg1)} {get_char_register(arg2)}")
    print_register(chunk)

def jmp(chunk):
    global a, b ,c ,d, s, i ,f,stack
    arg1, op, arg2 = chunk
    if not arg1 or (arg1 & f != 0):
        i = read_register(arg2)
        print(f"JMP TAKE JUMP TO {i:02x}")
        print_register(chunk)
        return
    print(f"JMP Not TAKE")  
    print_register(chunk)

def sys(chunk):
    global a, b ,c ,d, s, i ,f,stack
    arg1, op, arg2 = chunk
    if (arg1 & 1) != 0 :
        file = stack[a]
        writer_register(arg2, file)
        print("OPEN File")
        print_register(chunk)
        exit()
    
    if (arg1 & 4 ) != 0:
        v3 = c
        if ( 3*(256 - b) ) < v3:
            v3 = 3 * (256 - b )
        v4 = len(input)
        writer_register(arg2, v4)
        print("Read 3")
        print_register(chunk)
        
    
    if (arg1 & 2) != 0:
        v5 = c
        if (256 - b) < v5:
            v5 = -b & 0xff
        print(f"Read  {v5} bytes from file {a:02x} to memory stack[{b:02x}]")
        writer_register(arg2, v5)
        print_register(chunk)
    
    if (arg1 & 16) != 0:
        v7 = c
        if ( 256 - b) < c:
           v7 = -c
        print(f"Write {v7} bytes from memory[ {b:02x} to file {a:02x}]")
        writer_register(arg2, v7) 
        print_register(chunk)
    
    if (arg1 & 0x20) != 0:
        print(f"Exit {a}")
        exit()

if __name__ == '__main__':
    # 示例用法
    file_path = r"F:\研二上\coding\ctf\pwncollege\command"  # 替换为实际的文件路径
    a , b, c, d, s, i, f =0,0,0,0,0,0,0

    stack = [0] * 1024 # 0x300

    while True:
        chunk = read_binary_file_in_chunks(file_path, i)
        
        print("\n")
        print(f"Read chunk: {chunk.hex()}")
        i = i + 1
        arg1, op, arg2 = chunk
        if (op & 0x20) != 0:
            imm(chunk)
        if (op & 0x80) != 0:
            add(chunk)
        if (op & 0x02) != 0:
            stk(chunk)            
        if (op & 0x40) != 0:
            stm(chunk)
        if (op & 0x01) != 0:
            ldm(chunk)        
        if (op & 0x04) != 0:
            cmp(chunk)        
        if (op & 0x10) != 0:
            jmp(chunk)        
        if (op & 0x08) != 0:
            sys(chunk)    
    
    
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