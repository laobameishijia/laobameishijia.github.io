---
title: Vue学习-5-过渡和动画
date: 2021-11-16 09:25:00
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

# 过渡和动画

Vue.js过渡可以使页面元素在出现和消失时实现多种过渡效果。Vue在插入、更新或者移除DOM时，提供了多种方式的应用过渡效果。开发者可以使用transition组件,结合CSS的动画Animation、过渡Transition或者js来操作DOM使元素动起来。

## CSS过渡

![过渡实现过程](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211116183654.png)

```html
<!DOCTYPE html>
<html>
 <head>
  <meta charset="utf-8">
  <title></title>
  <style>
   .fade-enter, .fade-leave-to {
    opacity: 0
  }
  .fade-enter-active, .fade-leave-active {
    transition: opacity .5s
  }
  </style>
 </head>
 <body>

  <div id="demo">
   <button v-on:click="show = !show">
    切换按钮
   </button>
   <transition name="fade">
    <p v-if="show">hello</p>
   </transition>
  </div>
  <script src="../../js/vue.js"" type="text/javascript" charset="utf-8"></script>
  <script>
   new Vue({
    el: '#demo',
    data: {
     show: true
    }
   })
  </script>

 </body>
</html>

```

## CSS动画

CSS动画的用法与CSS过渡的用法相同，其区别是，在CSS动画中，v-enter类名在节点插入DOM后不会立即删除，而是在animationend事件触发时删除。

> 这个看的不是很懂，后面用到再说吧

## JS过渡

Js过渡是使用javascript钩子函数实现的过渡效果，这些钩子函数可以结合CSS的transition/animations使用，也可以单独使用。

感觉用js这个从结构上面看的话，就清楚很多了

```html
   <transition name="fade" @before-enter="beforeEnter" @enter="enter" @after-enter="afterEnter" @before-leave="beforeLeave"
    @leave="leave" @after-leave="afterLeave">
    <p v-show="show"></p>
   </transition>
```

![控制台输出](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211116185453.png)

## 案例--新增列表项的动画效果

具体的细节和效果用到的时候再去官网看就行了，还有很多细节和用法，就不在这赘述了。。。

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>
<style>
    li {
        border: 1px dashed #999;
        margin: 5px;
        line-height: 35px;
        padding-left: 5px;
        font-size: 12px;
        width: 100%;
    }

    li:hover {
        background-color: cornflowerblue;
        transition: all 1s ease;
    }

    .v-enter,
    .v-leave-to {
        opacity: 0;
        transform: translateY(80px);
    }

    .v-enter-active,
    .v-leave-active {
        transition: all 0.5s ease;
    }

    /* 下面的 .v-move 和 .v-leave-active 配合使用，能够实现列表后续的元素，渐渐地飘上来的结果 */
    .v-move {
        transition: all 0.5s ease;
    }

    .v-leave-active {
        position: absolute;
    }
</style>

<body>
    <div id="app">
        <div>
            <label>
                学号:
                <input type="text" v-model="id">
            </label>
            <label>
                姓名:
                <input type="text" v-model="name">
            </label>
            <input type="button" value="添加" @click="add">

        </div>
        <ul>
            <!-- 在实现列表过渡的时候，如果需要过渡的元素，是通过 v-for 循环渲染出来的，不能使用 transition 包裹，需要使用 transitionGroup -->
            <!-- 如果要为 v-for 循环创建的元素设置动画，必须为每一个元素设置：key 属性 -->
            <transition-group appear>
                <!-- 删除需要传入i -->
                <li v-for="(item, i) in list" :key="item.id" @click="del(i)">
                    {{ item.id }} --- {{ item.name }}
                </li>
            </transition-group>
        </ul>
    </div>
</body>

</html>
<script src="../../js/vue.js"" type=" text/javascript" charset="utf-8"></script>
<script>
    var vm = new Vue({
        el: '#app',
        data: {
            id: '',
            name: '',//这两个是为添加学号和姓名而设置的
            list: [
                { id: 1, name: '张三' },
                { id: 2, name: '李四' },
                { id: 3, name: '王五' },
                { id: 4, name: '赵四' }
            ]
        },
        methods: {
            add() {
                this.list.push({ id: this.id, name: this.name })
                this.id = this.name = ""
            },
            del(i) {
                // 从 i 的地方删，删除一个
                this.list.splice(i, 1)
            }
        }
    })
</script>

```
