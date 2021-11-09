---
title: Vue学习-2-Vue内置指令
date: 2021-11-8 09:25:00
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

# Vue.js内置指令

- **基本指令** `v-text`,`v-html`,`v-cloak`,`v-once`,`v-if`,`v-else`,`v-show`,`v-on`,`v-for`、数组更新
- **v-bind指令** 当数据变化时，可以对属性进行重新渲染。
- **v-model指令** 本质是监听用户的输入事件，从而更新数据。它会将Vue实例中的数据作为数据来源，当输入事件发生时，它会实时更新Vue实例中的数据，从而实现数据的双向绑定。

## 作业

> 实现一个简单的购物车

![效果](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211109094727.png)

### 思路

这个就是算个总和，没有什么难度。不过有一点需要注意。就是这个v-for的循环渲染的问题

![v-for循环渲染](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211109094823.png)

要么写成这样 `v-for="(item,index) in shopItems"`要么写成`v-for="item in shopItems"`

写成 `v-for="(item) in shopItems"`是没办法渲染的。 ❌

### 代码

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
    <script src="https://cdn.staticfile.org/vue/2.4.2/vue.js"></script>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@4.6.1/dist/css/bootstrap.min.css"
        integrity="sha384-zCbKRCUGaJDkqS1kPbPd7TveP5iyJE0EjAuZQTgFLD2ylzuqKfdKlfG/eSrtxUkn" crossorigin="anonymous">
</head>

<body>
    <div id="vue_det">
        <ul class="list-group list-group-horizontal">
            <li class="list-group-item  flex-fill">序号</li>
            <li class="list-group-item  flex-fill">商品名称</li>
            <li class="list-group-item  flex-fill">商品单价</li>
            <li class="list-group-item  flex-fill">购买数量</li>
            <li class="list-group-item  flex-fill">操作</li>
        </ul>
        <ul v-for="(item,index) in shopItems" class="list-group list-group-horizontal-sm">
            <li class="list-group-item flex-fill">{{item.id}}</li>
            <li class="list-group-item flex-fill">{{item.name}}</li>
            <li class="list-group-item flex-fill">{{item.price}}</li>
            <li class="list-group-item flex-fill">
                <button @click="addone(index)">+</button>
                {{item.number}}
                <button @click="reduceone(index)">-</button>
            </li>
            <li class="list-group-item flex-fill">
                <button @click="deleteone(index)">删除</button>
            </li>
        </ul>
        总计:{{allprice}}
    </div>
    <script src="https://cdn.jsdelivr.net/npm/jquery@3.5.1/dist/jquery.slim.min.js"
        integrity="sha384-DfXdz2htPH0lsSSs5nCTpuj/zy4C+OGpamoFVy38MVBnE+IbbVYUew+OrCXaRkfj"
        crossorigin="anonymous"></script>

    <script src="https://cdn.jsdelivr.net/npm/bootstrap@4.6.1/dist/js/bootstrap.bundle.min.js"
        integrity="sha384-fQybjgWLrvvRgtW6bFlB7jaZrFsaBXjsOMm/tB9LTS58ONXgqbR9W8oWht/amnpF"
        crossorigin="anonymous"></script>
    <script type="text/javascript">
        var vm = new Vue({
            el: '#vue_det',
            data: {
                shopItems: [
                    { id: '1', name: '苹果', price: 10, number: 0 },
                    { id: '2', name: '橘子', price: 20, number: 0 },
                    { id: '3', name: '香蕉', price: 11, number: 0 },
                    { id: '4', name: '橙子', price: 13, number: 0 },

                ],
                allprice: 0
            },
            methods: {
                addone(index) {
                    this.shopItems[index].number = this.shopItems[index].number + 1;
                    this.allprice = this.allprice + this.shopItems[index].price
                },
                reduceone(index) {
                    if (this.shopItems[index].number != 0) {
                        this.shopItems[index].number = this.shopItems[index].number - 1;
                        this.allprice = this.allprice - this.shopItems[index].price
                    }
                    else {
                        alert("该商品数量已经为零！")
                    }
                },
                deleteone(index) {
                    //价格的变化,要在删除这个选项之前
                    this.allprice = this.allprice - this.shopItems[index].price* this.shopItems[index].number
                    this.shopItems.splice(index, 1);
                }
            }
        })
    </script>
</body>

</html>
```