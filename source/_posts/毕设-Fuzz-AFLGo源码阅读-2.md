---
title: 毕设-Fuzz-AFLGo源码阅读-2
date: 2021-12-22 09:25:00
author: 美食家李老叭
img: https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211119142134.png
top: false
hide: false
cover: false
coverImg: 
password: 
toc: true
mathjax: false
summary: AFL框架的源码基本内容阅读
categories: 毕设
tags:
  - Fuzz
  - AFLGo
---

# 毕设-Fuzz-AFLGo源码阅读-2

## AFL框架

### 共享内存中的bitmap结构

TODO

### forkserver机制

TODO 


## Linux C

### read && write

看这个书，这个上面写的很详细

https://www.bookstack.cn/read/linux-c/c917e635f91d4a4f.md

### itimerval

 [参考](https://beachboyy.blog.csdn.net/article/details/35569229?spm=1001.2101.3001.6650.3&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-3.fixedcolumn&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-3.fixedcolumn)

```c++
struct itimerval {
    struct timeval it_interval; /* 计时器重启动的间歇值 */
    struct timeval it_value;    /* 计时器安装后首先启动的初始值 */
};
 
struct timeval {
    long tv_sec;                /* 秒 */
    long tv_usec;               /* 微妙(1/1000000) */
};
```

这段代码实现的功能：3秒钟后启动定时器，然后每隔1秒钟向终端打印count的递增值，当count到10时程序退出。

```c++
#include <stdio.h>
#include <time.h>
#include <sys/time.h>
#include <stdlib.h>
#include <signal.h>
 
 
static int count = 0;
 
 
void set_timer()
{
	struct itimerval itv;
 
 
	itv.it_value.tv_sec = 3;    //timer start after 3 seconds later
	itv.it_value.tv_usec = 0;
 
 
	itv.it_interval.tv_sec = 1;
	itv.it_interval.tv_usec = 0;
 
 
	setitimer(ITIMER_REAL,&itv,NULL);
}
 
 
void signal_handler(int m)
{
	count ++;
	printf("%d\n",count);
}
 
 
int main()
{
	signal(SIGALRM,signal_handler);
	set_timer();
	while(count < 10);
	exit(0);
	return 0;
 
}
```

### setsid()

setsid主要是重新创建一个session,子进程从父进程继承了SessionID、进程组ID和打开的终端,子进程如果要脱离父进程，不受父进程控制，我们可以用这个setsid命令

![setsid ping 127.0.0.1](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211223200534.png)

可以发现即使我们按下`ctrl+c` ping命令依然在执行，也就是说ping命令脱离了父进程shell的控制。

![ps -ef | grep ping](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211223200711.png)

### dup2

去看csdn这篇博客 [https://blog.csdn.net/silent123go/article/details/71108501](https://blog.csdn.net/silent123go/article/details/71108501)

从shell中运行一个进程，默认会有3个文件描述符存在(0、１、2)，0与进程的标准输入相关联，１与进程的标准输出相关联，2与进程的标准错误输出相关联，一个进程当前有哪些打开的文件描述符可以通过/proc/进程ID/fd目录查看。

### 管道

很清楚

https://www.bookstack.cn/read/linux-c/f72a2171d262cc79.md#783frj

### __builtin_expect

链接：https://www.jianshu.com/p/2684613a300f

```c++
#define likely(x) __builtin_expect(!!(x), 1) //x很可能为真       
#define unlikely(x) __builtin_expect(!!(x), 0) //x很可能为假

if(likely(value))  //等价于 if(value)
if(unlikely(value))  //也等价于 if(value)

example

上面的代码中 gcc 编译的指令会预先读取 y = -1 这条指令，这适合 x 的值大于 0 的概率比较小的情况。如果 x 的值在大部分情况下是大于 0 的，就应该用 likely(x > 0)，这样编译出的指令是预先读取 y = 1 这条指令了。这样系统在运行时就会减少重新取指了

int x, y;
 if(unlikely(x > 0))
    y = 1; 
else 
    y = -1;
```

## C

### fprintf

![fprintf](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211224190553.png)


## 汇编

### 零碎的知识

![mov指令](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220102144443.png)

[深入浅出GNU X86-64 汇编](https://www.cnblogs.com/lsgxeva/p/11176000.html)

[bss段，data段、text段、堆(heap)和栈(stack)](https://www.cnblogs.com/yanghong-hnu/p/4705755.html)

### XMM寄存器组

除了我们已经讨论过的寄存器，现代处理器还有一些扩展。这些扩展体现在电路上，指令集上，有时候也会扩展一些很有用的寄存器。比较著名的扩展叫作 SSE (Streaming SIMD Extensions)，该扩展加入了新的 xmm 寄存器集合：xmm0，xmm1，...，xmm15。这些寄存器为 128 位宽，常用于两种任务：

- 浮点数运算；以及
- SIMD 指令集(这种指令一条指令可以操作多条数据)
常用的 mov 指令没有办法操作 xmm 寄存器。movq 指令可以代替用来拷贝 xmm 寄存器的低位(128 位中的低 64 位)，操作数的其中一个可以也是 xmm 寄存器，或者通用寄存器，或者内存(也得是 64 位)。

为了填满 xmm 寄存器，你有两个选择：movdqa 和 movdqu。前者可以解释为“ move aligned double quad word”，移动两个对齐的 qword。后者是未对齐的版本。


