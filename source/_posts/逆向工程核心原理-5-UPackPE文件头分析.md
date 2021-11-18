---
title: 逆向工程核心原理-5-UPackPE文件头分析
date: 2021-11-17 09:25:00
author: 美食家李老叭
img: https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211111205602.png
top: false
hide: false
cover: false
coverImg: 
password: 
toc: true
mathjax: false
summary: PE文件格式
categories: 研究生预备学习
tags:
  - 逆向工程核心原理
---

# UPackPE文件头分析

## 分析UPack的PE头

### 重叠文件头

重叠文件头是其他压缩器经常使用的技法，借助该方法可以把MZ文件头(IMAGE_DOS_HEADER)与PE文件头(IMAGE_NT_HEADERS)巧妙的叠加在一起。

![重叠文件头的对比](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211118092257.png)  

MZ文件头(IMAGE_DOS_HEADER)中有以下两个重要成员。其余的成员对程序运行无意义

> offset(0) e_magic : Magic number = 4D5A('MZ')
> offset(3C) e_lfanew: File address of new exe header

问题在于PE文件格式规范，IMAGE_NT_HEADERS的起始位置是"可变的"，由e_lfanew来决定。

**正常的情况下**：`e_lfanew = MZ文件头大小(40) + DOS存根大小(可变：VC++下为A0) = E0`

这并不违反规定，只是钻了规范本身的空子

### IMAGE_FILE_HEADER.SizeOfOptionalHeader

![正常与修改后的对比](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211118093808.png)

从PE头文件来看，**IMAGE_OPTIONAL_HEADER**的起始偏移加上**SizeOfOptionalHeader**的值后才是**IMAGE_SECTION_HEADER**。增大**SizeOfOptionalHeader**以后，就相当于在**IMAGE_OPTIONAL_HEADER**与**IMAGE_SECTION_HEADER**之间增加了额外的空间，Upack就在这个区域增加解压代码。

![简要原理](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211118095011.png)

## 重叠节区

Upack重叠PE节区与文件头

![重叠示意图](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211118200644.png)


❓❓❓❓看的不是很明白,这里面尤其第一\第二节区, 它们确实是被重叠到Header上面了,但是内容上是怎么重叠的呢?肯定要删去一些东西把~~~

![解压后的第一个节区](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211118200732.png)

映射的话, 应该不算很难, 更改加载的虚拟地址就行了.

## RVA to RAW

利用了PE装载器发现第一个节区的PointerToRawData(10)不是FileAlignment(200)的整数倍时,它会强制将其识别为整数倍.(该情况下为0); 这样做的话, Upack文件就可以正常运行, 但是很多PE相关使用程序就会发生错误.

## 导入表

这个emm,也没看懂

![导入表的地址](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211118202538.png)


![第三个节区地址](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211118202409.png)


按照上面的将RVA->RAW

> RAW = RVA (271EE) - VirtualOffset(27000) + RawOffset(0)  = 1EE
> 注意: 3rd Section的RawOffset值不是10,而会强制变换为0

书上说该处就是Upack节区隐藏玄机的地方

![文件偏移IEE--第一个结构体](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211118203940.png)

上面所选区域就是IMAGE_IMPORT_DESCRIPTOR结构体组成的数组, 偏移IEE~201为第一个结构体, 其后既不是第二个结构体, 也不是NULL结构体

![3rd节区映射到内存](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211118203752.png)

----

❓❓❓❓ 为什么偏移200是第三个节区的结束呢? 之前的节区表里面, 写的是`RawSize为 1F0`

----

从文件看导入表好像是坏了,但是在加载到内存里面之后,看起来又是好的. 

## 导入地址表

通过上面结构体的数据, 得到


| 偏移 | 成员                   | RVA  |
| ---- | ---------------------- | ---- |
| 1EE  | OriginalFistThunk(INT) | 0    |
| 1FA  | Name                   | 2    |
| 1FE  | FirstThunk(IAT)        | 11E8 |

Name 的RVA是2, 它属于Header区域,因为第一个节区是从RVA 1000开始的.

![文件偏移2](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211118210327.png)

😱😱😱这样看的话,好像Upack把数据重复利用了?! 既可以是Kernel32.dll, 又可以代表其他含义?

FirstThunk(IAT) 转换为RAW是`IE8`

![IE8](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211118210552.png)

这部分区域就是IAT域, 同时也作为INT来使用. 也就是说该处是 `Name Pointer(RVA)`数组 RVA 28/BE, 其结束是NULL. RVA位置上存放着导入函数的 `Ordinal+名称字符串`

![RVA 28](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211118211013.png)

Q: 这样的话,导入函数最终的地址放到哪里去了?? 

A: 这只是压缩过的代码,最后还得解压缩,应该会回复成正常的样子的...
