
---
title: Writeup for 西湖论剑CTF
date: 2023-02-16 09:25:00
author: 美食家李老叭
img: https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20230216220007.png
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


# 西湖论剑CTF

这个比赛只做了一道题，看了很久确实也没做出来。没有想到程序的执行逻辑可以在32位和64位的不同执行环境下进行切换。

## Dual personality

总体来讲该程序 

- 第一次 从 `32-> 64 赋值key -> 32`

- 第二次 从 `32-> 64 循环左移 -> 32`

- 第三次 从 `32 -> 64 进行了两次加密，一次就是基本的逻辑运算，另一次是异或。-> 32`



1. 将exe程序送进ida，发现main函数无法被正常反编译。

![反编译main函数](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/1676538544590.png)

`sub_401120`函数会把下面的二进制修改为 `EA D0 11 40 00 33 00`-----`jmp far 33:004011D0h`往后面跳转之后程序就变成了执行64位语句了。所以需要把相应的二进制dump下来放到64位ida中进行分析。

> 这里不要被这个 `jmp far ptr loc_4014FE+2`给欺骗了，这个语句没有被正确解析。这句话会远跳转到33:4011D0这个地址。远跳转有一个隐形的操作，他会将代码段寄存器CS设置为跳转到的这个段对应的段选择子，这里执行完了远跳转之后CS的值会变为0x33，

![更改程序语句](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20230216171224.png)

![jmp far 33:004011D0h](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20230216171341.png)

2. 把4011d0h地址处的二进制dump出来用ida64分析之后。

![64位ida分析](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20230216173515.png)

> 注意 gs:qword_60 中 qword_60 是指 `60 00 00 00`而不是 0x00000060地址处的内容，否则人家会写成 `gs:[qword_60] `
> ![qword_60](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20230216202725.png)

3. 其实 `407000h`处存放的是原本 `call sub_401120`下面一句的地址 `004013E8`。当返回32位执行模式之后，刚好回到下面的语句。

这里有一个加密的过程。

![加密过程](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20230216195345.png)

4. 在加密完成之后，有一个 `call fword ptr` 其实和 `jmp far`是一样的道理切换到 64位模式下执行。

![call fword ptr](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20230216195947.png)

这次跳转的地址为 `00401200`, 跟之前的操作是一样的， 同样是dump下来放到ida中分析。

![00401200](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20230216202940.png)

注意这里面的 `0x40705c` -- 在第一次跳转到64模式下执行的过程中，将 `BeingDebugged`的标志位放到了这个地址里面。

> `__ROL8__` 是一个循环左移的操作 其中`retaddr[1]`指向的实际上是 `用户输入`。

执行完成之后返回到32位执行环境中。

5. 第三次跳转到64位环境里面

![第三次跳转](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20230216205121.png)

同样dump 0x401290的字节码，使用64位IDA分析，这个函数主要修改了0x407000保存的返回值，然后对0x407014开始 的数组中的常量进行处理，得到新的值。

![0x401290](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20230216205403.png)

**注意的是 407050h 存放的是`0x401467`**

**注意：前两次切换到64位模式后，函数结束就重新切换到32位模式了，但是这次出来后仍然为64位模式，因此main函数之后的代码也都是64位的指令**

![32位下](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20230216213201.png)

![64位下](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20230216213059.png)

> 此时 `407000h`中存放的实际上是 `0x004014C5`
![第三次跳转时已经修改了407000h中的存放的地址](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20230216213842.png)

6. 后面就是常见的比较字符串的操作了。

![跟密文进行比较](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20230216215312.png)



## 参考资料

1. win PEB 中BeingDebugged标志位获取---https://blog.csdn.net/Simon798/article/details/100702034

2. MK_FP -- https://baike.baidu.com/item/MK_FP

3. ret 和 retf 的区别 -- https://zhuanlan.zhihu.com/p/372398363

4. 西湖论剑官方RE-Writeup  ----https://mp.weixin.qq.com/s?__biz=MzU1MzE3Njg2Mw==&mid=2247504667&idx=1&sn=ce9b15211cf7a9658e96697a14f361f7&chksm=fbf4496bcc83c07d1c704d81479c4fc75169661fe98d58f73d61b92af3579da64d3f9dcba810&mpshare=1&scene=23&srcid=0213j43MPfxQ7G87C9gjkRQh&sharer_sharetime=1676276996609&sharer_shareid=029344dcf9624c2d719bc3d553ed4aa3#rd