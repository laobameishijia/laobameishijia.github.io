---
title: PWN-College-Writeup
date: 2024-06-17 11:13:00
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

![20240617111433](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20240617111433.png)

这一关跟上一关二进制程序的思路都是一样的，唯一不同的是，取消了回显。你无法直接通过回显判断各个指令、寄存器、系统调用对应的特征数。

首先想到的是，爆破，遍历所有的可能的指令表示。但很明显这个空间太大了，需要先缩小空间。于是，我根据各个指令操作寄存器的特征，尝试出不存在对应寄存器的特征值是`0x01`,`IMM`指令对应的特征数是`0x01`,`SYS`指令对应的特征数是`0x40`,`exit`系统调用对应的特征数是`0x02`。

**判断的思路是，指令有8种特征值，而寄存器只对应其中7种，恰好IMM指令只操作一种寄存器，所以可以根据不对应寄存器的特征值判断出IMM指令。然后剩下的指令中只有SYS指令只操作一种寄存器，所以也可以尝试出SYS指令。然后只有exit系统调用没有涉及寄存器，根据0x01这个不对应寄存器的特征数可以判断出exit指令。**

判断出能判断的指令之后，剩下的代码中尚未判断的指令、寄存器、系统调用有STM指令、a,b,c寄存器、open,read,write系统调用。最后，就是在可能的空间中遍历所有可能。

必须得感谢GPT的代码，这大大减少了我的工作量。不过虽然自动化代码减轻了负担，还是要尽可能了解代码书写的思路。

## 尝试

由于程序在最初会读取 `flag`文件的内容去生成种子，但是它没有关闭这个文件描述符`fd`。所以起初，为了缩小爆破空间，我想利用这个文件描述符，这样的话，不用重新将`/flag`字符串压入到栈中。因为文件描述符的赋值是按照从小到大进行赋值的，而标准输入、输出、以及报错占用了1，2，3。我又检查了程序其他位置没有打开文件的情况，所以，我猜测它应该是4。

可是没有成功，因为前期读取`flag`文件的时候，`fd`的文件偏移已经被置为文件末尾的位置了。所以再次通过这个文件描述符读取文件内容时，已经没有内容可以读取了。这个时候，有两种方式重置，第一种使用`lseek函数`将文件描述符的偏移量重新定位到文件开头，第二种重新打开文件，获取新的文件描述符，然后从新的文件描述符中读取内容。

第一种，我们无法实现；第二种，跟爆破一样。所以此方法失败。

```python
# b'\x01\x04\{a}',  # imm a = 0x04
# b'\x01\x60\{c}',  # imm c = 0x60  
# b'\x01\x20\{b}',  # imm b = 0x20  
# b'\x40\{a}\{read}',  # read(a, b, c) --return bytes -> a
# b'\x01\x60\{c}',  # imm c = 0x60   
# b'\x01\x01\{a}',  # imm a = 0x01 
# b'\x01\x20\{b}',  # imm b = 0x20
# b'\x40\{a}\{write}',  # write(a, b, c) ---return bytes -> a
# b'\x40\x10\x02'  # exit

```

## 知识

### 文件描述符

在C程序中，文件描述符是由操作系统分配的整数值，用于标识进程打开的文件或其他输入/输出资源（如管道、套接字等）。文件描述符的分配遵循以下规律：

预定义的文件描述符：

标准输入（stdin）: 文件描述符 0
标准输出（stdout）: 文件描述符 1
标准错误（stderr）: 文件描述符 2
顺序分配：

当一个进程打开一个新的文件或资源时，操作系统会按照从低到高的顺序分配文件描述符。通常，它会分配当前未使用的最小整数值。
例如，如果一个进程最初只打开了标准输入、标准输出和标准错误，那么下一个打开的文件将获得文件描述符 3。
文件描述符的重用：

如果一个文件描述符被关闭（例如通过 close 函数），那么该文件描述符的值可以被重新分配给以后打开的文件或资源。


### 父子进程之间的继承



#### 如果通过其他程序启动这个程序，fd的值会有所不同吗？

ChatGPT
是的，如果通过其他程序启动这个程序，文件描述符（fd）的值可能会有所不同。这是因为文件描述符的分配是由操作系统管理的，并且是针对每个进程独立进行的。

当一个新进程启动时，它会继承其父进程的一些文件描述符（如标准输入、标准输出、标准错误）。此外，如果父进程在启动子进程之前已经打开了其他文件或资源，这些文件描述符也可能会传递给子进程。因此，子进程中的文件描述符分配可能会因为父进程的状态而有所不同。

具体示例
假设你有一个父进程 parent_program，它启动子进程 flag_program：
```c
flag_program.c

#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <assert.h>

unsigned __int64 flag_seed() {
    unsigned int seed;
    unsigned int i;
    int fd;
    __int64 buf[17];
    unsigned __int64 v5;

    v5 = __readfsqword(0x28u); // 获取当前栈指针，具体实现视系统而定
    memset(buf, 0, 128);
    fd = open("/flag", O_RDONLY);
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

int main() {
    unsigned __int64 seed = flag_seed();
    printf("Seed: %llu\n", seed);

    return 0;
}
```
```c
parent_program.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
    // 父进程打开一个文件，这会占用一个文件描述符
    int fd = open("parent_file.txt", O_RDONLY);
    if (fd < 0) {
        perror("Error opening parent_file.txt");
        return 1;
    }

    // 启动子进程
    pid_t pid = fork();
    if (pid == 0) {
        // 子进程
        execl("./flag_program", "./flag_program", NULL);
        perror("Error executing flag_program");
        return 1;
    } else if (pid > 0) {
        // 父进程
        wait(NULL);
    } else {
        perror("Error forking");
        return 1;
    }

    close(fd);
    return 0;
}
```
运行结果分析
在上述例子中：

父进程 parent_program 在启动子进程 flag_program 之前打开了一个文件 parent_file.txt，这个文件描述符可能是 3（如果没有其他文件描述符被使用）。
子进程 flag_program 继承了父进程的文件描述符表，因此在子进程中，标准输入（0）、标准输出（1）、标准错误（2）之后，下一个可用的文件描述符将是 4。
因此，如果父进程打开了额外的文件或资源，子进程中的 fd 值将会受到影响。例如，如果 parent_program 打开了一个文件描述符 3，子进程 flag_program 中打开的 /flag 文件描述符可能会是 4 而不是 3。

总结
文件描述符的分配是从未使用的最小整数开始的。
如果父进程打开了额外的文件或资源，子进程会继承这些文件描述符，这会影响子进程中新分配的文件描述符的值。
在不同的环境和上下文中启动同一程序，文件描述符的具体值可能会有所不同。

### 权限


1. 子进程的权限继承 ---- 这一点比较重要！
当父进程启动子进程时，子进程通常会继承父进程的权限（用户ID和组ID）。因此，如果父进程本身没有权限访问 flag 文件，那么通过父进程启动的子进程也不会有权限访问 flag 文件。（调试程序的原理恐怕类似）

2. 程序的权限提升（SUID）
如果 baby 程序是一个具有权限提升（Set User ID, SUID）的程序，并且该程序的所有者具有读取 flag 文件的权限，那么子进程在运行 baby 程序时可以获取到 flag 文件的内容。

## 解题代码

```python
import itertools
import subprocess

data_list_template = [

    b'\x01\x10\{a}', # IMM a = \x10
    b'\x01\x2f\{b}', # IMM b = \x2f
    b'\{stm}\{b}\{a}', # stm *a = b

    b'\x01\x11\{a}', # IMM a = \x11
    b'\x01\x66\{b}', # IMM b = \x66
    b'\{stm}\{b}\{a}', # stm *a = b

    b'\x01\x12\{a}', # IMM a = \x12
    b'\x01\x6c\{b}', # IMM b = \x6c
    b'\{stm}\{b}\{a}', # stm *a = b
    
    b'\x01\x13\{a}', # IMM a = \x13
    b'\x01\x61\{b}', # IMM b = \x61
    b'\{stm}\{b}\{a}', # stm *a = b

    b'\x01\x14\{a}', # IMM a = \x14
    b'\x01\x67\{b}', # IMM b = \x67
    b'\{stm}\{b}\{a}', # stm *a = b
    
    b'\x01\x15\{a}', # IMM a = \x15
    b'\x01\x00\{b}', # IMM b = \x00
    b'\{stm}\{b}\{a}', # stm *a = b

    b'\x01\x00\{b}', # IMM b = \x00
    b'\x01\x10\{a}', # IMM a = \x10
    b'\x40\{a}\{open}', # open  -- return file -> a

    b'\x01\x60\{c}', # imm c = 0x60  
    b'\x01\x20\{b}', # imm b = 0x20  
    b'\x40\{a}\{read}', # read(a, b, c) --return bytes -> a


    b'\x01\x60\{c}', # imm c = 0x60   
    b'\x01\x01\{a}', # imm a = 0x01 
    b'\x40\{a}\{write}', # write  ---return bytes -> a

    b'\x40\x10\x02' # exit

    # b'\x01\x04\{a}',  # imm a = 0x04
    # b'\x01\x60\{c}',  # imm c = 0x60  
    # b'\x01\x20\{b}',  # imm b = 0x20  
    # b'\x40\{a}\{read}',  # read(a, b, c) --return bytes -> a
    # b'\x01\x60\{c}',  # imm c = 0x60   
    # b'\x01\x01\{a}',  # imm a = 0x01 
    # b'\x01\x20\{b}',  # imm b = 0x20
    # b'\x40\{a}\{write}',  # write(a, b, c) ---return bytes -> a
    # b'\x40\x10\x02'  # exit
]

# op_num = [0x02, 0x04, 0x08, 0x10, 0x20, 0x80]
op_num = [0x20, 0x80]

register_num = [0x02, 0x04, 0x08, 0x10, 0x20, 0x40, 0x80]
sys_num = [0x01, 0x04, 0x08, 0x10, 0x20, 0x40, 0x80]

# 
op_combinations = list(itertools.permutations(op_num, 1))

# Generate all possible unique combinations for registers a, b, and c
register_combinations = list(itertools.permutations(register_num, 3))

# Generate all possible unique combinations for sys read \ write \ open
sys_combinations = list(itertools.permutations(sys_num, 3))

def format_data_list(a, b, c, read, write, stm, open_):
    formatted_data_list = []
    for item in data_list_template:
        formatted_item = item.replace(b'\{a}', bytes([a]))
        formatted_item = formatted_item.replace(b'\{b}', bytes([b]))
        formatted_item = formatted_item.replace(b'\{c}', bytes([c]))

        formatted_item = formatted_item.replace(b'\{read}', bytes([read]))
        formatted_item = formatted_item.replace(b'\{write}', bytes([write]))
        formatted_item = formatted_item.replace(b'\{open}', bytes([open_]))

        formatted_item = formatted_item.replace(b'\{stm}', bytes([stm]))
        formatted_data_list.append(formatted_item)
    return formatted_data_list

# Set timeout in seconds
timeout = 0.1

# Iterate over all combinations
for op_combo in op_combinations:
    for reg_combo in register_combinations:
        for sys_combo in sys_combinations:
            stm, = op_combo
            a, b, c = reg_combo
            read, write, open_ = sys_combo
            formatted_data_list = format_data_list(a, b, c, read, write, stm, open_)
            
            # Convert list of byte strings to a single byte string for subprocess input
            data_input = b''.join(formatted_data_list)

            try:
                # Run the external program with the data_input and timeout
                process = subprocess.Popen(
                    ['/challenge/babyrev_level22.1'],
                    stdin=subprocess.PIPE,
                    stdout=subprocess.PIPE,
                    stderr=subprocess.PIPE
                )
                
                stdout, stderr = process.communicate(input=data_input,timeout=timeout)

                # Print output (for debugging purposes)
                print(f"Operation: stm={stm}, Registers:  a={a}, b={b}, c={c}, sys: read={read}, write={write}, open={open_}")
                stdout = stdout.decode()
                if("pwn" in stdout):
                    print(f"Output: {stdout}")
                    exit(1)
                if stderr:
                    print(f"Error: {stderr.decode()}")
            
            except subprocess.TimeoutExpired:
                print(f"Operation: stm={stm}, Registers: a={a}, b={b}, c={c}, sys: read={read}, write={write}, open={open_} - Timed out")
                process.kill()
                stdout, stderr = process.communicate()
                # print(f"Output after timeout: {stdout.decode()}")
                if stderr:
                    pass
                    # print(f"Error after timeout: {stderr.decode()}")

```
