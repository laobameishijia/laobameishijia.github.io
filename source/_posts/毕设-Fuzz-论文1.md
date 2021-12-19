---
title: 毕设-Fuzz-论文1
date: 2021-12-9 09:25:00
author: 美食家李老叭
img: https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211119142134.png
top: false
hide: false
cover: false
coverImg: 
password: 
toc: true
mathjax: false
summary: Directed Greybox Fuzzing----AFLGo框架的论文
categories: 毕设
tags:
  - Fuzz
---

# Directed Greybox Fuzzing

第一次完全从头看到尾的十六页的论文, 一方面是为了清楚AFLGo实现的原理, 另一方面是为开题报告积累一些材料. 后面, 还需要再把介绍AFL框架的论文看一遍. 因为AFLGo是以其为基础进行扩展的. 那想要了解地更深入, 就不可避免地要把基础搞懂.


这篇论文读下来, 发现AFLGo是在AFL的基础上添加了模拟退火算法. 其他的,因为看论文的时间过去了一段时间, 居然就都忘记了.

AFL重要的设计思想因为没有论文只能是去看项目中的txt, 很多东西也是看不太懂. 只能慢慢地硬啃了......