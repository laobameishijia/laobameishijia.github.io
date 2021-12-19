---
title: 毕设-Fuzz-AFLGo源码阅读-1
date: 2021-12-15 09:25:00
author: 美食家李老叭
img: https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211119142134.png
top: false
hide: false
cover: false
coverImg: 
password: 
toc: true
mathjax: false
summary: 阅读AFLGo源码
categories: 毕设
tags:
  - Fuzz
  - AFLGo
---

# AFLGo源码阅读-1

首先还是要从基础的C语言语句开始补起。

## C 知识

### C 预处理

C 预处理器不是编译器的组成部分，但是它是编译过程中一个单独的步骤。简言之，C 预处理器只不过是一个文本替换工具而已，它们会指示编译器在实际编译之前完成所需的预处理。我们将把 C 预处理器（C Preprocessor）简写为 CPP。

所有的预处理器命令都是以井号（#）开头。它必须是第一个非空字符，为了增强可读性，预处理器指令应从第一列开始。下面列出了所有重要的预处理器指令：

![预处理指令列表](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211215211258.png)

其他的详见菜鸟教程。
[https://www.runoob.com/cprogramming/c-preprocessors.html](https://www.runoob.com/cprogramming/c-preprocessors.html)

### C/C++ 中 volatile 关键字详解

C/C++ 中的 volatile 关键字和 const 对应，用来修饰变量，通常用于建立语言级别的 memory barrier。这是 BS 在 "The C++ Programming Language" 对 volatile 修饰词的说明：

> A volatile specifier is a hint to a compiler that an object may change its value in ways not specified by the language so that aggressive optimizations must be avoided.

volatile 关键字是一种类型修饰符，用它声明的类型变量表示可以被某些编译器未知的因素更改，比如：操作系统、硬件或者其它线程等。遇到这个关键字声明的变量，编译器对访问该变量的代码就不再进行优化，从而可以提供对特殊地址的稳定访问。声明时语法：int volatile vInt; 当要求使用 volatile 声明的变量的值的时候，**系统总是重新从它所在的内存读取数据**，即使它前面的指令刚刚从该处读取过数据。而且读取的数据立刻被保存。例如:

```c
volatile int i=10;
int a = i;
...
// 其他代码，并未明确告诉编译器，对 i 进行过操作
int b = i;
```

volatile 指出 i 是随时可能发生变化的，每次使用它的时候必须从 i的地址中读取，因而编译器生成的汇编代码会重新从i的地址读取数据放在 b 中。而**优化**做法是，由于编译器发现两次从 i读数据的代码之间的代码没有对 i 进行过操作，它会自动把上次读的数据放在 b 中。而不是重新从 i 里面读。这样以来，如果 i是一个寄存器变量或者表示一个端口数据就容易出错，所以说**volatile 可以保证对特殊地址的稳定访问**。



## 概念知识

### 程序处理中的控制流图和调用图

#### 控制流图--Control Flow Graph

控制流图(Control Flow Graph, CFG)也叫控制流程图，是一个过程或程序的抽象表现，是用在编译器中的一个抽象数据结构，由编译器在内部维护，代表了一个程序执行过程中会遍历到的所有路径。它用图的形式表示一个过程内所有基本块执行的可能流向, 也能反映一个过程的实时执行过程。Frances E. Allen于1970年提出控制流图的概念。此后，控制流图成为了编译器优化和静态分析的重要工具。

原文：
In a control-flow graph each node in the graph represents a basic block, i.e. a straight-line piece of code without any jumps or jump targets; jump targets start a block, and jumps end a block. Directed edges are used to represent jumps in the control flow. There are, in most presentations, two specially designated blocks: the entry block, through which control enters into the flow graph, and the exit block, through which all control flow leaves.
译文：
在控制流图中，图中的每个节点代表一个基本块，即一段没有任何跳转或跳转目标的直线代码；跳转目标开始一个块，而跳转结束一个块。有向边用来表示控制流中的跳转。在大多数演示中，有两个特别指定的块：入口块，控制通过它进入流程图；出口块，所有控制流通过它离开。

![控制流图的几种结构](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211219105935.png)

**特点**:

- 控制流程图是过程导向的
- 控制流程图显示了程序执行过程中可以遍历的所有路径
- 控制流程图是一个有向图
- CFG 中的边描述控制流路径，节点描述基本块
- 每个控制流图都存在2个指定的块：Entry Block(输入块)，Exit Block(输出块)

#### 函数调用图 Function Call Graph

![函数调用图](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211219111655.png)

