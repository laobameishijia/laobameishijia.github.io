---
title: Vue学习-1-Vue入门
date: 2021-11-6 09:25:00
author: 美食家李老叭
img: https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211106094659.png
top: false
hide: false
cover: false
coverImg: 
password: 
toc: true
mathjax: false
summary: 基本的入门
categories: 研究生预备学习
tags:
  - Vue
---

# 第一个Vue.js应用

有些东西没必要都详细的列出来，列个清单知道学了啥。不会再去查就行。

- **模板** 可以渲染指定的内容到挂载的位置
- **数据** 双向绑定，数据发生变化。视图也跟着发生变化
- **方法** methods中定义 `{{say()}}`引用
- **观察|监听** watch选项可以监听数据变化
- **数据绑定** 插值`{{}}`、表达式绑定`{{complete?'完成':'未完成'}}`、双向数据绑定`v-model`
- **计算属性** vue实例中的一个选项
- **生命周期** 看起来是跟浏览器渲染的顺序过程有关系

## 作业

> 实现一个简单的计算器

### 思路

主要就是数据结构运算思路,由前缀表达式转换为后缀表达式，在通过后缀表达式进行运算。

因为是简单的计算器嘛，就十以内的加减乘除(**不带括号的那种**)。😊😊😊

参考的博客[http://blog.csdn.net/antineutrino/article/details/6763722](http://blog.csdn.net/antineutrino/article/details/6763722)

### 代码

例子 1+2*3+1

```js
  var vm = new Vue({
   el: '#vue_det',
   data: {
    one: "1",
    equation: "",
    result: "hello",
    // 后缀表达式
    op: [],//运算符栈
    nm: [],//操作数栈
    // 判断运算符优先级
    priorHigher: function (a, b) {
     return (a === '+' || a === '-') && (b === '*' || b === '/');
    },
   },
   methods: {
    number: function () {
     var str = event.target.value;
     this.equation = this.equation + str;
    },
    operator: function () {
     var str = event.target.value;
     this.equation = this.equation + str;
    },
    // 判断运算符优先级
    priorHigher: function (a, b) {
     return (a === '+' || a === '-') && (b === '*' || b === '/');
    },
    // 中缀表达式转后缀
    calculate: function () {
     //初始化
     this.op=[];
     this.nm=[];
     for (var i = 0; i < this.equation.length; i++) {
      tag = false
      //如果是数字就直接压栈
      if (!isNaN(this.equation[i])) {
       this.nm.push(this.equation[i])
      }
      //如果是运算符
      else {
       while (!tag) {
        //运算符栈中为空，就直接压
        if (this.op.length == 0) {
         this.op.push(this.equation[i])
         tag = true
        }
        else {
         var a = this.op.pop()
         this.op.push(a)
         //优先级比栈顶的高，那就压栈
         if (this.priorHigher(a, this.equation[i])) {
          this.op.push(this.equation[i])
          tag = true
         }
         //否则，弹出栈顶压入nm栈，再进行循环
         else {
          this.op.pop()
          this.nm.push(a)
         }
        }
       }
      }
     }
     var fortag = this.op.length
     for (var i = 0; i < fortag; i++) {
      var a = this.op.pop()
      this.nm.push(a)
     }
     alert(this.nm)//简单的计算器给弄好了 Yes good
     for (var i = 0; i < this.nm.length; i++) {
      //如果是数字就直接压栈
      if (!isNaN(this.nm[i])) {
       this.op.push(this.nm[i])
      }
      //如果遇到了运算符,弹出栈顶的两个元素做对应的运算，再把结果压进去
      else{
       var a = parseInt(this.op.pop())
       var b = parseInt(this.op.pop())
       if(this.nm[i]==='+') this.op.push(b+a)
       if(this.nm[i]==='-') this.op.push(b-a)
       if(this.nm[i]==='*') this.op.push(b*a)
       if(this.nm[i]==='/') this.op.push(b/a)
      }
     }
     alert(this.op)
    },

   }
  })

```
