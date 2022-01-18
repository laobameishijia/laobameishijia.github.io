---
title: 毕设-Fuzz-AFLGo源码阅读-3
date: 2022-1-2 09:25:00
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

# 毕设-Fuzz-AFLGo源码阅读-3

## AFL框架

### 共享内存中的bitmap结构 && forkserver机制

不太好描述，就直接放图上来了。  ---其他的都写到注释里面了

![共享内存中的bitmap结构&&forkserver机制](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220118215515.jpg)


## Linux C

### 进程之间的信号是如何进行传递的

直接看下面的链接把

[https://www.bookstack.cn/read/linux-c/5016547c13b140cc.md#6b906b](https://www.bookstack.cn/read/linux-c/5016547c13b140cc.md#6b906b)

### 控制台和终端的关系

[https://www.cnblogs.com/sparkdev/p/11460821.html](https://www.cnblogs.com/sparkdev/p/11460821.html)

在计算机里，把那套直接连接在电脑上的键盘和显示器就叫做控制台。而终端是通过串口连接上的，不是计算机自身的设备，而控制台是计算机本身就有的设备，一个计算机只有一个控制台。计算机启动的时候，所有的信息都会显示到控制台上，而不会显示到终端上。这同样说明，控制台是计算机的基本设备，而终端是附加设备。计算机操作系统中，与终端不相关的信息，比如内核消息，后台服务消息，都可以显示到控制台上，但不会显示到终端上。比如在启动和关闭 Linux 系统时，我们可以在控制台上看到很多的内核信息(下图来自 vSphere Client 中的 "Virtual Machine Console")

![控制台](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220103170553.png)

现在终端和控制台都由硬件概念，逐渐演化成了软件的概念。**简单的说，能直接显示系统消息的那个终端称为控制台，其他的则称为终端(控制台也是一个终端)。或者我们在平时的使用中压根就不区分 Linux 中的终端与控制台。**

![Linux上的终端](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220103171552.png)

### /dev/urandom /dev/null

一个是随机数生成器, 另一个相当于空文件, 所有定向到这个地方的输入都会消失。

