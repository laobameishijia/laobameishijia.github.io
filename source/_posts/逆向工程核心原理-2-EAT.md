---
title: 逆向工程核心原理-2-EAT
date: 2021-11-14 09:25:00
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

# EAT

Windows操作系统中，“库”是为了方便其他程序调用而集中包含相关函数的文件(DLL/SYS)。Win32 API是最具代表性的库，其中的kernel32.dll文件被称为最核心的库文件。

EAT是一种核心机制，它使不同的应用程序可以调用库文件中提供的函数。也就是说，只有通过EAT才能准确求得相应库中导出函数的起始地址。

![EAT](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211114094826.png)

从库中获得函数地址的API为`GetProcAddress`函数。该API引用EAT来获取指定API的地址。

## GetProcAddress操作原理

1. 利用AddressOfName成员转到“函数名称数组”

![函数名称数组](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211114101014.png)

2. 函数名称数组中存储着字符串的地址。通过比较字符串，查找指定的函数名称(此时的数组索引成为name_index)

![找到函数名称](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211114101732.png)

3. 利用AddressOfNameOrdinals成员转到orinal数组。

![转到orinal数组](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211114101919.png)

4. 在ordinal数组中通过name_index查找到相应的ordinal值

![找到ordinal的值](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211114102139.png)

5. 利用AddressOfFunctions成员转到“函数地址数组EAT”

![转到EAT](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211114102324.png)

6. 利用刚刚求到的ordinal用作数组索引，获得指定函数的起始地址。

![获得起始地址](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211114102451.png)

> kernel32.dll中所有导出函数均有相应名称，AddressOfNameOrdinals数组的值以index=ordinal的形式存在。但并不是所有的DLL文件都如此，导出函数中也有一些函数没有名称。

---
> 对于没有函数名称的导出函数，可以通过ordinal查找到它们的地址。从Ordinal值中减去IMAGE_EXPORT_DIRECTORY.base成员后得到一个值。使用该值作为“函数地址数组”索引，即可查找到相应函数的地址。
---

