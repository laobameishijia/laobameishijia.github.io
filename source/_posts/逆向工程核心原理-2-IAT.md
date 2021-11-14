---
title: 逆向工程核心原理-2-IAT
date: 2021-11-13 09:25:00
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

# PE文件格式

从DOS头到节区头是PE头部分，其下的节区合称为PE体。文件中使用offset，内存中使用VA(Virtual Address)虚拟地址来表示位置。文件加载到内存时，情况就会发生变化(节区的大小、位置等)。文件的内容一般可分为代码`.text`、数据`.data`、资源`.rsrc`。

![PE文件](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211113151027.png)

## PE头

PE头由许多结构体组成。

- DOS头
- DOS存根
- NT头
  - 文件头
  - 可选头
- 节区头

RVA(Relative Vritual Address) to RAW(文件偏移地址是指数据在PE文件中的地址，是文件在磁盘上存放时相对于文件开头的偏移。文件偏移地址从pe文件的第一个字节开始计数，起始值为0)

`RAW - PointerToRawData = RVA - VirtualAddress`

PointerToRawData: 磁盘文件中节区的起始位置
VirtualAddress: 内存中节区的起使地址

### IAT

Import Address Table导入地址表，IAT是一种表格，用来记录程序正在使用哪些库中的哪些函数。

#### DLL

16位的DOS不存在DLL，只有库Library一说，比如在C语言中使用printf()函数时，编译器会先从C库中读取相应函数的二进制代码，然后插入应用程序。也就是说，可执行文件中包含着printf函数的二进制代码。Windows OS支持多任务，若仍采用这种包含库的方式会变得非常没有效率。在同时运行多个程序的情况下，会造成严重的资源浪费(内存和磁盘空间)。因此设计出了DLL概念：

- 不需要把库包含在程序中，单独组成DLL文件，需要时调用即可
- 内存映射技术使加载后的DLL代码、资源在多个进程中实现共享
- 更新库时只需要替换相关的DLL文件即可

>如何理解DLL文件节约了磁盘和内存空间？
之前的多个程序无论是在源代码还是加入内存时，都包含了printf函数的二进制代码。现在大家可以共享内存中DLL文件中的printf函数，这样的话，不仅源代码中不用包含printf函数(节约了磁盘空间)，加载至内存后还可以共享一个printf函数(节约了内存空间)

#### IMAGE_IMPORT_DESCRIPTOR(IMPORT Directory Table)

记录PE文件要导入哪些库文件
![IMAGE_IMPORT_DESCRIPTOR](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211113160018.png)

```text
IAT输入顺序
1. 读取IID(IMAGE_IMPORT_DESCRIPTOR)的Name成员，获取库名称字符串(kernel32.dll)
2. 装载相应库 -> LoadLibrary("kernel32.dll")
3. 读取OriginalFirstThunk成员获取INT地址(Import Name Table)
4. 逐一读取INT中的数组的值，获取相应的IMAGE_IMPORT_BY_NAME地址(RVA)
5. 使用IMAGE_IMPORT_BY_NAME的Hint或Name获取相应函数的起始地址 -> GetProcAddress("GetCurrentThreadld")
6. 读取IID的FirstThunk(IAT)成员，获取IAT地址
7. 将上面获取到的函数地址输入相应的IAT数组值
8. 重复以上步骤4-7，直到INT结束(遇到NULL时)
```

`IMAGE_IMPORT_DESCRIPTOR`结构体数组不在PE头而在PE体中，但查找其位置的信息在PE头中，`IMAGE_OPTIONAL_HEADER32.DataDirectory[1].VirtualAddress`即是`IMAGE_IMPORT_DESCRIPTOR`结构体数组的起始地址(RVA值)。**期间注意RVA和RAW(文件偏移)之间的转换--要用到节区头中.text端的相关信息**

>整体上寻找信息的思路为
> 1. 先在 `IMAGE_OPTIONAL_HEADER32`中找到 `IMPORT Directory`的RVA值并将其转换成`RAW文件偏移`，并根据转换出来的文件偏移找到`IMPORT Directory`的位置
> ![寻找IMPORT Directory](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211113173647.png)
> 2. 再根据`IMPORT Directory`相应数据(**同样要转换为RAW文件偏移**)找到导入函数表、导入DLL表、导入函数的实际地址表。
> ![寻找其他表](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211113174053.png)

`IMAGE_IMPORT_DESCRIPTOR`结构体中的命名很奇怪，OriginalFirstThunk对应的是**导入函数表**INT(Import Name Table ) address。FirstThunk对应的是**导入函数地址表**IAT(Import Address Table)address。Name对应的是 **导入DLL表**library name string address