---
title: 毕设-Fuzz-3
date: 2021-11-22 09:25:00
author: 美食家李老叭
img: https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211119142134.png
top: false
hide: false
cover: false
coverImg: 
password: 
toc: true
mathjax: false
summary: 毕设是模糊测试相关的,提前了解
categories: 毕设
tags:
  - Fuzz
---
# NoteBook阅读

## SearchBasedFuzzer

我们先不去对代码实现细节进行掌握, 先去掌握思想.

什么是基于搜索的测试? 为什么要用这种方式? 原话: ***Sometimes we are not only interested in fuzzing as many as possible diverse program inputs, but in deriving specific test inputs that achieve some objective, such as reaching specific statements in a program.*** 要产生特定的测试数据,从而到达程序中特定的位置.

这样的方式需要我们做哪些工作?

- 首先,你要明确你要生成的数据类型和范围. *Maybe XML\String\Int etc. a-z\1-10*
- 其次, 要定义在搜索空间内的适应度函数, 也就是你要能评价搜索到的数据距离目标的距离
- 然后, 定义搜索算法. 即按照什么样的方式\以何种顺序在搜索空间中搜素.
  - **Hillclimbing** 搜素范围规模不大.***个人感觉根梯度下降非常相似,只不过在这里是离散的.***
  - **Genetic Algorithm**搜素范围规模较大.***结合了自然选择和种群进化的生物理论.***

### 两个主要的算法

The **hillclimbing algorithm** itself is very simple: 

1. Take a random starting point
2. Determine fitness value of all neighbours
3. Move to neighbour with the best fitness value
4. If solution is not found, continue with step 2

The **GA emulates natural evolution** with the following process:

- Create an initial population of random chromosomes
- Select fit individuals for reproduction
- Generate new population through reproduction of selected individuals
- Continue doing so until an optimal solution has been found, or some other limit has been reached.

> 当然在具体的应用中, 还要设计到如何设计突变/如何选择种群/如何利用父代产生子代. 这些都需要跟实际情况相结合.

### 需要了解的一些背景知识

Python 源码到机器码的过程，以 CPython 为例，编译过程如下：

- 将源代码解析为解析树（Parser Tree）
- 将解析树转换为抽象语法树（Abstract Syntax Tree）
- 将抽象语法树转换到控制流图（Control Flow Graph）
- 根据流图将字节码（bytecode）发送给虚拟机（ceval）

可以使用以下模块进行操作：

- ast 模块可以控制抽象语法树的生成和编译
- py-compile 模块能够将源码换成字节码（编译），保存在 __pycache__ 文件夹，以 .pyc 结尾（不可读）
- dis 模块通过反汇编支持对字节码的分析（可读）

