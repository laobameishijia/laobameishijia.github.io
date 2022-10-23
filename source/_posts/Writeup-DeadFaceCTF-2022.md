---
title: Writeup for DeadFaceCTF2022
date: 2022-10-16 09:25:00
author: 美食家李老叭
img: https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221016125700.png
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
# Rev

这次rev做出了两道题，没有什么困难的地方，通过反编译软件可以很清楚地看到源码。程序主题的逻辑也很清晰，主要是涉及了一些关于AES和异或的加密过程。

## deadface2022_re05

这个主逻辑也很清楚 先要求你输入一个字符串，然后对这个字符串进行sha3哈希，然后判断这个hash字符串和程序中保存的正确的hash字符串一不一样。如果一样的话，他会通过这个字符串来解密(异或操作)加密的字符串，最后返回正确的flag。

解密的操作是通过hashcat来解密的，我看视频他用的是`3070ti`。当然还有一个字典，然后一下子就算出来了。

![hash破解](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221019184046.png)

## re03-another-fine-product

这是一个软件，通过软件中的点击操作获取到相应的信息。

**此题和现实中的软件破解很类似，为了保护软件的收益，通常会采取用户验证的方式。而这部分验证的代码通常是非常复杂且不可见的。但是有些时候，我们可以修改程序流，使程序跳过验证的步骤，从而达到破解软件的效果。**

### 主要逻辑

软件本身没有什么问题，就是简单的一些菜单和点击之后出现的弹窗。

![软件截图](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221023144355.png)

### 做法

![WindbgPreview](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221023144745.png)

### 基础的知识

- user32.dll Windows用户界面相关应用程序接口 user32.dll是Windows用户界面相关应用程序接口，用于包括Windows处理，基本用户界面等特性，如创建窗口和发送消息。
- ntdll.dll Windows NT内核级文件 ntdll.dll是重要的Windows NT内核级文件。描述了windows本地NTAPI的接口。当Windows启动时，ntdll.dll就驻留在内存中特定的写保护区域，使别的程序无法占用这个内存区域。


# PWN

这个题是`jack`与我交流的时候讨论到的。里面有一个非常重要的点需要总结。

以前我认为`call function_name`的时候，`function_name`必须要位于整个函数最开始的地方。

但实际上，并不是如此。因为CPU也不知道你这个`function_name`到底是不是在整个函数最开始的地方。

## 主体逻辑

1. 首先程序是运行在服务器端利用tcp套接字来与客户端进行交互，无任何保护措施，提供了上图的6个功能。
![主题逻辑](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221016131314.png)

2. `Make Target`功能，会通过用户的输入。利用malloc函数申请一个0x18的堆空间创建一个Target结构体。这个结构体的第二个成员是一个函数指针，指向一个删除结构体的函数。
   
![delete_structure](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221016131713.png)

![Target结构体](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221016131744.png)

3. `Create an Exploit`功能，会通过用户输入。利用malloc函数申请一个0x18的堆空间创建Exploit结构体。

![Exploit结构体](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221016131925.png)

4. `retrieve_admin_key`，验证用户输入的密码，正确则返回flag。

```C++
ssize_t retrieve_admin_key()
{
  ssize_t result; // eax

  send(fork_sock, "What is the password?\n", 0x17u, 0);
  recv(fork_sock, "aaaaaaaaaaaaaaaa", 0x10u, 0);
  if ( !strcmp("ga*wi58Fw#o&WOG9", "aaaaaaaaaaaaaaaa") )
    result = send(fork_sock, "flag{--------------------------}\n", 0x22u, 0);
  else
    result = send(fork_sock, "Bad Password\n", 0xEu, 0);
  return result;
}
```

## 堆利用

Target结构体和Exploit结构体都是申请的堆空间上大小为0x18的空间。而且程序在这个过程中没有申请过其他堆块。此外delete_struct的函数里面的只是简单调用了`free`函数。`free`函数会操作堆块的头部，而不会清除此前保留在堆块中的数据。指向堆块的指针实际上还是指向的这个已经被释放的堆块的用户可以用空间首地址。但是这个时候注意，由于堆块已经释放，此前用户可用地址的前8个字节已经变成了指向前一个可用堆块和后一个可用堆块的指针。

那么，我们通过创建一个Target然后再删除，然后创建一个`Exploit`结构体，这样的话，Exploit结构体申请到的堆空间与之前删除的Target结构体申请的堆空间是一致的。那么Delete_structure这个成员的内容就变成我们可以操控的了。接着就需要使用delete功能调用这个函数就行。

```c++
   switch ( buf )
    {
      case 0:
      case 49:
        v5 = newS1(v7);
        break;
      case 50:
        if ( v5 ) // v5在调用delete函数之后，实际上还是指向的已经被释放的堆空间的地址。
        //所以，我们再exploit中构造函数指针之后，依然是可以调用我们修改的函数指针。
          (*(void (__cdecl **)(int, int *))(v5 + 16))(v5, v7);
        else
          send(*v7, "No target?\n", 0xCu, 0);
        break;
      case 51:
        v6 = newS2(v7);
        break;
      case 52:
        check_compatability(v5, v6, v7);
        break;
      case 53:
        retrieve_admin_key();
        break;
      default:
        return __readgsdword(0x14u) ^ v9;
    }
```

![对比](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221016133405.png)

该程序没有开启任何的保护机制，所以我们接下来要考虑的就是应该修改或者跳转到哪里？

开始我一直以为，这个地址只能是函数的首地址。实际上通过我们最开始描述，这个地址不是首地址也可以。只不过可能后续程序就没办法正常运行了。

所以我们直接跳到了下图所示位置，注意这个时候要保证send函数参数的正确性。注意寄存器里面的值前后的变化。

![跳转位置](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221016134322.png)

实际上，这个跳转位置并不是check_compatability函数的开始部分。因为我们无法知道这个正确的密码，只能是跳过验证的部分。

![效果](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221016134429.png)

## 细节

Target结构体申请到的堆空间

![Target结构体申请到的堆空间](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221016134658.png)

创建Target结构体之后堆空间的内容

![创建Target结构体之后堆空间的内容](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221016141528.png)

删除掉我们申请的Target时
![delete_structure](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221016135032.png)

删除掉Target之后，堆空间中的内容

![删除Target之后的堆块](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221016141136.png)

创建Exploit结构体

![创建Exploit结构体](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221016141808.png)

第二次，使用删除功能，可以发现之前指向堆空间的指针依然是指向的堆空间的用户可用地址。并没有被清零。实际只是堆块放到了空闲管理的堆块表里面去了。而且我们调用的call也只指向了retrieve_admin_key函数

![第二次，使用删除功能](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221016141940.png)

当我们步入之后发现，call指令同样还是先把返回地址压栈，然后再将我们设置的指令地址弹出到eip中。
![第二次，使用删除功能](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221016142156.png)

之后我们就能显示出flag。

![效果](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221016142446.png)

## 知识点

### linux 堆块结构体

```C++
/*
  This struct declaration is misleading (but accurate and necessary).
  It declares a "view" into memory allowing access to necessary
  fields at known offsets from a given base. See explanation below.
*/
struct malloc_chunk {

  INTERNAL_SIZE_T      prev_size;  /* Size of previous chunk (if free).  */
  INTERNAL_SIZE_T      size;       /* Size in bytes, including overhead. */

  struct malloc_chunk* fd;         /* double links -- used only if free. */
  struct malloc_chunk* bk;

  /* Only used for large blocks: pointer to next larger size.  */
  struct malloc_chunk* fd_nextsize; /* double links -- used only if free. */
  struct malloc_chunk* bk_nextsize;
};
```

   - prev_size, 如果该 chunk 的物理相邻的前一地址 chunk（两个指针的地址差值为前一 chunk 大小）是空闲的话，那该字段记录的是前一个 chunk 的大小 (包括 chunk 头)。否则，该字段可以用来存储物理相邻的前一个 chunk 的数据。这里的前一 chunk 指的是较低地址的 chunk 。
  - size ，该 chunk 的大小，大小必须是 2 * SIZE_SZ 的整数倍。如果申请的内存大小不是 2 * SIZE_SZ 的整数倍，会被转换满足大小的最小的 2 * SIZE_SZ 的倍数。32 位系统中，SIZE_SZ 是 4；64 位系统中，SIZE_SZ 是 8。 该字段的低三个比特位对 chunk 的大小没有影响，它们从高到低分别表示
  NON_MAIN_ARENA，记录当前 chunk 是否不属于主线程，1 表示不属于，0 表示属于。
  IS_MAPPED，记录当前 chunk 是否是由 mmap 分配的。
  PREV_INUSE，记录前一个 chunk 块是否被分配。一般来说，堆中第一个被分配的内存块的 size 字段的 P 位都会被设置为 1，以便于防止访问前面的非法内存。当一个 chunk 的 size 的 P 位为 0 时，我们能通过 prev_size 字段来获取上一个 chunk 的大小以及地址。这也方便进行空闲 chunk 之间的合并。
  - fd，bk。 chunk 处于分配状态时，从 fd 字段开始是用户的数据。chunk 空闲时，会被添加到对应的空闲管理链表中，其字段的含义如下
  fd 指向下一个（非物理相邻）空闲的 chunk
  bk 指向上一个（非物理相邻）空闲的 chunk
  通过 fd 和 bk 可以将空闲的 chunk 块加入到空闲的 chunk 块链表进行统一管理
  - fd_nextsize， bk_nextsize，也是只有 chunk 空闲的时候才使用，不过其用于较大的 chunk（large chunk）。
  fd_nextsize 指向前一个与当前 chunk 大小不同的第一个空闲块，不包含 bin 的头指针。
  bk_nextsize 指向后一个与当前 chunk 大小不同的第一个空闲块，不包含 bin 的头指针。
  一般空闲的 large chunk 在 fd 的遍历顺序中，按照由大到小的顺序排列。这样做可以避免在寻找合适 chunk 时挨个遍历。

### CALL指令

CALL（LCALL）指令执行时，进行两步操作：
（1）将程序下一条指令的位置的IP压入堆栈中；
（2）转移到调用的子程序。