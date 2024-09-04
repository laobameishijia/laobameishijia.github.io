---
title: PWN-College-Web-2-Writeup
date: 2024-09-04 11:02:00
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

https://pwn.college/intro-to-cybersecurity/talking-web/

## 有价值的问题

### 1. starce工具

starce用于追踪和现实一个进程或多个进程的系统调用。

### 2. as与nasm的区别？

`as` 和 `nasm` 是两种不同的汇编器，它们使用不同的语法和指令集。以下是它们的主要区别：

#### 1. **语法风格**

- **`as` (GNU Assembler)**:
  - **语法风格**: `as` 默认使用 AT&T 语法。
  - **寄存器**: 在 AT&T 语法中，寄存器名前面带有 `%` 符号。例如，`%eax`、`%ebx`。
  - **操作数顺序**: AT&T 语法中，操作数的顺序是 `源, 目的`。例如，`mov %eax, %ebx` 将 `eax` 的值移动到 `ebx`。
  - **常量**: 常量前面带有 `$` 符号。例如，`$1` 表示常量 `1`。
- **`nasm` (Netwide Assembler)**:
  - **语法风格**: `nasm` 使用 Intel 语法。
  - **寄存器**: 在 Intel 语法中，寄存器名不带任何前缀。例如，`eax`、`ebx`。
  - **操作数顺序**: Intel 语法中，操作数的顺序是 `目的, 源`。例如，`mov ebx, eax` 将 `eax` 的值移动到 `ebx`。
  - **常量**: 常量直接写出，不需要前缀。例如，`1` 表示常量 `1`。

#### 2. **伪指令和宏**

- **`as`**:
  - 使用 `.section` 来定义数据段和代码段。
  - 使用 `.equ` 来定义常量。
  - 支持更复杂的伪指令和宏指令。
- **`nasm`**:
  - 使用 `section` 来定义数据段和代码段。
  - 使用 `equ` 来定义常量。
  - 提供了一些不同的宏功能和伪指令，比如 `%define`。

#### 3. **目标文件格式**

- `as`:
  - `as` 默认生成 ELF 格式的目标文件 (`.o`)，但可以通过选项指定其他格式。
- `nasm`:
  - `nasm` 可以生成多种格式的目标文件，包括 ELF、COFF、PE、Mach-O 等，可以通过 `-f` 选项指定，例如 `nasm -f elf file.asm`。

#### 4. **系统调用和调用约定**

- `as`:
  - 在 Linux 下，`as` 的语法更接近 Unix 系统原生使用的汇编代码，尤其是涉及系统调用、链接和调试时。
- `nasm`:
  - `nasm` 的 Intel 语法更加简洁易读，尤其对于编写裸机代码或 Windows 下的程序。

#### 5. **编译和链接流程**

- **`as`**:
  - 通常与 `gcc` 或 `ld` 配合使用，直接生成目标文件。
  - 例子：`as source.s -o source.o` 然后 `ld source.o -o output`。
- **`nasm`**:
  - 可以生成各种格式的目标文件，需指定格式。
  - 例子：`nasm -f elf32 source.asm -o source.o` 然后 `ld -m elf_i386 source.o -o output`。

#### 总结

`as` 和 `nasm` 都是强大的汇编器，但它们适用于不同的场景和风格。`as` 更适合与 GNU 工具链配合使用，尤其在 Linux 系统编程中；`nasm` 则以其简洁的 Intel 语法受到许多编写裸机代码和跨平台开发者的青睐。选择哪一个主要取决于你的项目需求和个人偏好。

### 3. 汇编语言的中包含系统调用，在ld进行编译的时候，都会链接什么文件？

在使用 `ld` 对包含系统调用的汇编语言代码进行编译时，链接器会将你的汇编代码与一些标准的系统库和启动代码（startup code）进行链接。以下是 `ld` 链接时通常涉及的文件和步骤：

#### 1. **启动代码 (Startup Code)**

- 启动代码

  是在程序开始执行前，由操作系统加载并调用的一段代码。它负责初始化程序的执行环境，如设置堆栈、清除数据段、调用构造函数等。启动代码的典型文件是：

  - **`crt1.o`** (C runtime): 这个文件包含程序的入口点代码 (`_start`)，它最终会调用程序的 `main` 函数（如果是 C 代码）或直接开始执行汇编代码。
  - **`crti.o` 和 `crtn.o`**: 这些文件包含初始化和终止代码，它们会在程序开始执行之前和程序结束时运行。

#### 2. **系统调用接口**

- **系统调用 (syscall) 接口** 是由内核提供的，用于程序与操作系统交互。它们通常是由汇编语言直接调用的，或通过 C 标准库进行封装。
- 如果你在汇编代码中使用了系统调用，例如 `int 0x80`（对于旧版 x86 Linux 系统），你直接与内核交互，不需要额外链接什么特别的库文件。
- **标准 C 库 (`libc.so`)**: 如果你使用了标准库中的函数（如 `printf`），则会链接到标准 C 库。这通常是动态链接库。

#### 3. **动态链接器 (Dynamic Linker)**

- 如果你链接了动态库，例如 `libc.so`，那么链接器会在最终的可执行文件中嵌入指向动态链接器的引用（如 `/lib/ld-linux.so.x`），它会在程序启动时运行，加载动态链接库。

#### 4. **标准库文件**

- **libc.so**: 标准 C 库（动态或静态），包含大量常用函数和系统调用封装。
- **ld-linux.so.x**: 动态链接器，在运行时加载和链接动态库。
- **libgcc.a** 或 **libgcc_s.so**: GCC 的运行时支持库，提供某些平台相关的低级支持功能。

#### 5. **符号表 (Symbol Table)**

- 链接器会解析所有未定义的符号，确保它们能在已链接的文件中找到定义。如果找不到符号定义，链接器会报错（未定义符号错误）。

#### 6. **链接器脚本 (Linker Script)**

- 链接器脚本控制链接器的行为。默认情况下，`ld` 使用一个标准链接器脚本，定义了内存布局和段分配，但你也可以编写自定义的链接器脚本来控制细节。

#### 总结

当你使用 `ld` 来链接包含系统调用的汇编程序时，主要会链接以下内容：

1. **启动代码文件** (`crt1.o`, `crti.o`, `crtn.o`) - 初始化程序的执行环境。
2. **系统调用接口或标准库** (`libc.so`, `libgcc_s.so`) - 如果你使用了标准库中的函数。
3. **动态链接器** (`ld-linux.so.x`) - 在运行时加载和链接动态库。
4. **符号解析** - 确保所有符号在链接的库或对象文件中被正确解析。

如果你的汇编程序直接调用系统调用，而没有依赖标准库函数，那么链接器可能只需要处理启动代码，而不需要链接到标准库。

### 4. AT&T语法和Intel语法的区别

#### 1. **操作数顺序**

在 AT&T 语法中，指令的源操作数在前，目标操作数在后。例如：

- **Intel 语法**: `MOV EAX, EBX` （将 `EBX` 的值移动到 `EAX` 中）
- **AT&T 语法**: `movl %ebx, %eax` （将 `%ebx` 的值移动到 `%eax` 中）

#### 2. **寄存器和内存操作数的前缀**

AT&T 语法中，寄存器名称以 `%` 开头，而内存操作数的寄存器名称以 `(%reg)` 的形式出现。

- **寄存器**: `%eax`, `%ebx`, `%ecx`, `%edx`（32位寄存器）
- **内存**: `(%eax)` 表示内存地址由 `eax` 寄存器指定

#### 3. **指令后缀**

指令的后缀表示操作数的大小（字节、字、双字、四字）。常见的后缀包括：

- `b` 表示字节（8位）
- `w` 表示字（16位）
- `l` 表示双字（32位）
- `q` 表示四字（64位）

例如：

- `movb %al, %bl` （8位寄存器之间移动数据）
- `movw %ax, %bx` （16位寄存器之间移动数据）
- `movl %eax, %ebx` （32位寄存器之间移动数据）
- `movq %rax, %rbx` （64位寄存器之间移动数据）

#### 4. **立即数和寄存器**

立即数（常数值）在 AT&T 语法中以 `$` 开头。例如：

- `movl $5, %eax` （将常数值 5 移动到 `%eax` 中）

#### 5. **内存地址模式**

AT&T 语法中的内存地址模式使用 `base + index*scale + displacement` 形式。例如：

- `movl 8(%ebp), %eax` （将从 `%ebp` 寄存器偏移 8 字节的内存值移动到 `%eax` 中）

###

###

## level 6

这个地方在建立连接之后，一直报错，结果是客户端发送过来的请求包的长度大于了服务器设置的缓冲器的长度。实际发送的长度是146，我第一开始设置的缓冲区的大小只是100

```bash
===== Trace: Parent Process =====
[✓] execve("/proc/self/fd/3", ["/proc/self/fd/3"], 0x7fed1c3f0a10 /* 0 vars */) = 0
[✓] socket(AF_INET, SOCK_STREAM, IPPROTO_IP) = 3
[✓] bind(3, {sa_family=AF_INET, sin_port=htons(80), sin_addr=inet_addr("0.0.0.0")}, 16) = 0
[✓] listen(3, 0)                            = 0
[✓] accept(3, NULL, NULL)                   = 4
[✓] read(4, "GET / HTTP/1.1\r\nHost: localhost\r\nUser-Agent: python-requests/2.32.3\r\nAccept-Encoding: gzip, deflate, zstd\r\nAccept: */*\r\nConnection: keep-alive\r\n\r\n", 1000) = 146
[✓] write(4, "HTTP/1.0 200 OK\r\n\r\n", 19) = 19
[✓] close(4)                                = 0
[✓] exit(0)                                 = ?
[?] +++ exited with 0 +++
```

```asm
.section .data
domain:    .int 2                # AF_INET
type:      .int 1                # SOCK_STREAM
protocol:  .int 0                # IPPROTO_IP
socketfd:  .quad 0               # socket_fd
response:  .asciz "HTTP/1.0 200 OK\r\n\r\n"
buffer:    .space 1000  # 就是这里，第一开我设置的是100

.section .text
.global _start

_start:
    # 创建套接字
    movl $2, %edi                # 第一个参数: domain (AF_INET)
    movl $1, %esi                # 第二个参数: type (SOCK_STREAM)
    movl $0, %edx                # 第三个参数: protocol (IPPROTO_IP)
    movl $41, %eax               # 系统调用号: socket (41 in x86_64)
    syscall                      # 触发系统调用

    # 保存套接字描述符
    movq %rax, %rdi              # 将返回的文件描述符保存在rdi中
    movq %rax, socketfd(%rip)          # 将文件描述符保存在socketfd中

    # 准备sockaddr_in结构
    movq $16, %rdx               # 第三个参数: struct sockaddr的长度
    leaq sockaddr(%rip), %rsi    # 第二个参数: struct sockaddr指针

    # 绑定套接字
    movl $49, %eax               # 系统调用号: bind (49 in x86_64)
    syscall                      # 触发系统调用

    # 监听
    movq $0, %rsi               # listen(socket_fd, max_listen_num)
    movq $0x32, %rax             # 系统调用号 listen
    syscall                      # 触发系统调用

    # 创建连接
    movq $0x2B, %rax            # accept(socket_fd, address, addrlen)
    xor %rdx, %rdx
    xor %rsi, %rsi
    syscall

    # 读取响应
    movq %rax, %rdi
    leaq buffer(%rip), %rsi    # read(fd, response, size(response))
    movq $1000, %rdx
    movq $0, %rax
    syscall

    # 写入响应
    leaq response(%rip), %rsi    # write(fd, response, size(response))
    movq $19, %rdx
    movq $1, %rax
    syscall
    
    # 关闭这个套接字
    movq $3, %rax
    syscall

    # 退出
    movl $60, %eax               # 系统调用号: exit (60 in x86_64)
    xorl %edi, %edi              # 返回代码0
    syscall                      # 触发系统调用

# sockaddr_in 结构体
sockaddr:
    .short 0x0002                     # 地址族 (AF_INET)
    .short 0x5000                # 端口号 (端口号80)
    .long  0x00000000                     # IP地址 (INADDR_ANY, 绑定到任意地址)
    .quad  0                     # 填充以达到16字节对齐

```

## 