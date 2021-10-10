---
title: 汇编语言学习-[bx]和loop指令
date: 2021-10-10 09:25:00
author: 美食家李老叭
img: 
top: false
hide: false
cover: false
coverImg: 
password: 
toc: true
mathjax: false
summary: loop循环指令和[bx]段内偏移的综合应用
categories: 研究生预备学习
tags:
  - 汇编
---

# 总结

## [bx]

`mov ax,[bx]`, bx中存放的数据作为一个偏移地址EA,段地址SA默认在ds中,将SA:EA处的数据送入ax中,**注意这里是字型数据哦！** 即:`ax = ds*16 + bx`

建议以后再写汇编语言程序的时候,把[bx]前面的段寄存器显式地标注出来,也就是所谓的**段前缀**。

## loop指令

loop 指令就是一个循环指令，注意cx循环次数,和bx在循环过程中的变化。

## 综合应用

计算ffff:0-ffff:b单元中的数据的和,结果存储在dx中

### 数据相加的问题

- dx = dx + 内存中的8位数据 类型不匹配
- dl = dl + 内存中的8位数据 结果越界

解决方案：利用一个16位的寄存器来做中介。将内存单元中的8位数据赋值到一个16位寄存器ax中**高八位要初始化为0**,再将ax中的数据加到dx上,从而使两个运算对象的类型匹配并且结果不会超界。

## 实验

### 1 and 2

编程 ,向内存 0:200~0:23F依次传送数据0~63

```text
  assume cs:codeseg

  codeseg segment

  start: mov ax,0200H    ;这里第一次写成0H了 心里想的确实是0200 不知道怎么弄成0了
        mov ds,ax
      
        mov bx,0H
        mov cx,40H
      
  s:  mov ds:[bx],bl
      inc bx
      loop s
      
      mov ax,4c00H
      int 21h
  codeseg ends
  end start
  end
```

### 3

将"mov ax,4c00h"之前的指令复制到内存0:200h处,补全程序,上机调试,跟踪运行结果.

```text
  assume cs:codeseg

  codeseg segment

  start: mov ax,cs
      mov ds,ax

      mov ax,0020h
      mov es,ax
      mov bx,0
      mov cx,17h;第一次写的21(10进制),看网上有说18的，感觉不对呀，我17的话就刚刚好是可以复制完的
      ;还有view里面cpu指令前面的地址是该指令的起始地址 你还要加上这个指令的大小,才算是下一条指令的相对地址,而且别忽略了最初的地址是从零开始算的

  s:  mov al,ds:[bx];标签表示的是相对于段定义起始位置的位置
      mov es:[bx],al
      inc bx
      loop s

      mov ax,4c00h
      int 21h
  codeseg ends
  end start
  end
```

![运行结果](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211010160654.png)