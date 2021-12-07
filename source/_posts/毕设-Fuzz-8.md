---
title: 毕设-Fuzz-8
date: 2021-12-6 09:25:00
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

# Parser

最近看的部分都是语法部分的, 从开始我们通过**语法来表述各种语言**. 到后面**用设计好的语法来生成字符串.**

但是呢, 其实也可以反过来. 就是先给一个字符串, 用这个字符串分解成语法部分---也就是前面所说的`the derivation tree of that string`.然后我们再用这个tree去生成其他的测试数据.

> 不过看了后面的例子, 他的意思是这样的. 我先已经有一个语法了, 但是呢, 这个语法的效果不是很好. 那么我们就需要从样本数据中提取模板. 然后在反过头去修改我们的语法. 这样可以使产生的数据更符合我们的要求. 换句话说, 就是有效的数据占比要更大.

**这部分后面的代码就不看了. 我觉得我当下的重点, 也并不是完全读懂所有的代码逻辑. 而是清楚并了解fuzz相关的背景知识, 然后再去针对性的看论文和代码. 这样效率会更高一点吧.**

------
看到后面这个部分的时候, 我突然感觉对于之前语法结构的理解有一些偏差.

`noterminal` `terminal` `symbol`之间的关系和区别

原文中是这么说:

In the above expression, the rule `<expr>` : `[<expr> + <expr>, <expr> - <expr>, <integer>] ` corresponds to how the nonterminal `<expr>` might be expanded. The expression `<expr> + <expr>` corresponds to one of the alternative choices. We call this an alternative expansion for the nonterminal `<expr>`. Finally, in an expression `<expr> + <expr>`, each of `<expr>`, `+`, and `<expr>` are symbols in that expansion. A symbol could be either a nonterminal or a terminal symbol based on whether its expansion is available in the grammar.

-----

