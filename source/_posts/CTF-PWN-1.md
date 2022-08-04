---
title: CTF-PWN-1
date: 2022-08-4 00:00:00
# 文章出处名称 #
from: 
# 文章出处链接 #
url: 
# 文章作者名称 #
author: 老叭美食家
# 文章作者签名 #
about: 不要因为没有掌声，就放弃你的梦想。
# 文章作者头像 #
avatar: 
# 是否开启评论 #
comments: true
# 文章标签 #
tags: CTF
# 文章分类 #
categories: PWN
# 文章摘要 #
description: PWN入门
---
# pwn

## 先检查保护机制

![保护机制](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220522095648.png)

开启了Nx和Relro保护，stack还是可以溢出，但是在stack中写shellcode的方式已经不可以了。因为Nx把数据所在的区域全部标记为不可执行的了。

## 查看反编译之后的代码

![反汇编](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220608101306.png)

观察到溢出点，`gets(v4 ,argv)`同时也可以看到程序中引入了system的系统调用，和相应的"/bin/sh"的字符串，所以我们可以通过ROP的方式解题。

ROP，Return-oriented programming面向返回导向式编程，会借助ret和栈顶，实现控制流的导向。

## 关键问题

system函数需要一个参数 `/bin/sh`，不能直接让system函数的地址覆盖掉 `返回地址`。需要利用ROP，将 `/bin/sh`的地址导入到 `rdi`寄存器里面，再调用system函数才行。

这个思路在第一开始，我是明白的。但是我不知道如何将 `/bin/sh`的地址保存到rdi寄存器里面去。还有就是，不知道这个system的地址到底应该是哪个？是extern里面的，还是plt里面的，亦或是 `_system`这个函数的？

## PLT原理

原理的部分，还是要补上。主要就是CSAPP的第七章。慢慢补上就好啦~~

# 解决思路

![ROP](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220715161804.png)

---
