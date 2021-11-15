---
title: Vue学习-4-过滤器和自定义指令
date: 2021-11-15 09:25:00
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

# 过滤器和自定义指令

Vue.js过滤器本质上就是一个函数，其作用是在用户输入数据后，**对数据进行处理**--`处理成我们想要的样子`，并返回一个处理结果。

## 过滤器的注册和使用

**形式一--全局**:

```html
<!DOCTYPE html>
<html>
 <head>
  <meta charset="utf-8">
  <title></title>
 </head>
 <body>
  <div id="app">
   <input type="text" name="" id="" value="" v-model.number="val"/>
   {{val|currencyDisplay}}
  </div>
  <script src="../../js/vue.js"" type="text/javascript" charset="utf-8"></script>
  <script type="text/javascript">
   Vue.filter('currencyDisplay', function(val) {
   return '$'+val.toFixed(2)
   },
   );
   new Vue({
    el:'#app',
    data:{
     val:5.35353
    }
   })
  </script>
  

 </body>
</html>

```

**形式二--局部**:

```html
 <body>
  <div id="app">
   <input type="text" name="" id="" value="" v-model="name" />
   <h1>{{ name | Upper }}</h1>
  </div>
  <script src="../../js/vue.js"" type="text/javascript" charset="utf-8"></script>
  <script type="text/javascript">
   var app = new Vue({
    el: '#app',
    data() {
     return {
      name: 'zhonghui'
     }
    },
    // 声明一个本地的过滤器
    filters: {
     Upper: function(value) {
      return value.toUpperCase()
     },
    }
   })
  </script>

 </body>
```

**形式三--叠加**

filterA被定义为接受单个参数的过滤器函数，表达式message的值将作为参数传递到函数中。继续调用同样被定义为接受单个函数的过滤器函数filterB，便可将filterA的结果传递到filterB中。

```html
{{message|filterA|filterB}}
```

同样也可以定义为接受多个参数的过滤器函数

```html
<!DOCTYPE html>
<html>
 <head>
  <meta charset="utf-8">
  <title></title>
  <style type="text/css">
   #app ul{
    width: 400px;
    background-color: lavender;
    list-style: none;
    padding: 10px;
    margin: 0;
   }
   #app h2{
    color:red;
    font-size: 16px;
   }
   .more{
    padding-left: 300px;
   }
   .more span{
    display: block;
    width: 65px;
    height:25px;
    padding: 5px;
    background-color: #006600;
    color: #fff;
    margin-top: 10px;
   }
  </style>
 </head>
 <body>
  <div id="app">
   <ul>
    <li v-for="article in articles">
     <h2> {{article.title}} 　</h2>
     <div class="summary"> {{article.summary|readMore(100, '...')}} </div>　　　　
     <div class="more"><span>阅读更多</span></div>
    </li>
   </ul>
  </div>
  <script src="../../js/vue.js"" type="text/javascript" charset="utf-8"></script>
  <script type="text/javascript">
   Vue.filter('readMore', function(text, length, suffix) {
    return text.substring(0, length) + suffix
   })
   let app = new Vue({
    el: '#app',
    data() {
     return {
      articles: [{
       title: 'Vue.js过滤器',
       summary: '过滤器本质上就是个函数，其作用在于用户在输入数据后，它能够进行处理，并返回一个处理结果。Vue.js提供了过滤器API，可以对数据进行过滤处理，并根据过滤的条件最终返回需要的结果。本章将会带你学习过滤器的注册及使用方法。',
      }]
     }
    }
   })
  </script>
 </body>
</html>
```

## 动态参数

感觉就是个动态显示

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <script src="../../js/vue.js"" type="text/javascript" charset="utf-8"></script>
    <title>Title</title>
</head>
<body>
<div id="app">
    <input type="text" v-model="price">
    <p>{{date|dynamic(price)}}</p>
</div>
<script type="text/javascript">
    Vue.filter('dynamic',
        function (date, price) {
            return date.toLocaleDateString() + " : " + price;
        });
    var vm = new Vue({
        el: '#app',
        data: {
            date: new Date(),
            price: 150
        }
    })
</script>
</body>
</html>
```

## 自定义指令的注册和使用

自定义指令是用来操作DOM的，尽管Vue推崇数据驱动视图的理念，但是并非所有情况都适合数据驱动理念。自定义指令就是一种有效的补充和扩展,其不仅可用于定义任意DOM操作，还可以复用。

### 自定义全局指令

自定义全局指令使用了`Vue.directive(指令ID，定义对象)`。定义对象是一个对象，这个对象上有一些指令相关的钩子函数，这些函数可以在特定的阶段执行相关的操作。

```html
<!DOCTYPE html>
<html>
 <head>
  <meta charset="utf-8">
  <title></title>
 </head>
 <body>
  <div id="app">
   <h3 v-red>使用自定义指令改变颜色</h3>
  </div>
  <script src="../../js/vue.js" type="text/javascript" charset="utf-8"></script>
  <script type="text/javascript">
   Vue.directive('red', function(el) {
    el.style["color"] = "red"
   })
   var vm = new Vue({
    el: '#app',
    data: {
     
    }
   });
  </script>
 </body>
</html>


```

## 钩子函数

除update与componentUpdated外，每个钩子函数都含有el、binding、vnode这三个参数。参数el就是指令绑定的DOM元素，而binding是一个对象，它包含name、value、oldvalue、expression、arg、modifiers等属性。除el之外，binding、vnode属性都是制度的。

| 名称             | Description  |
| ---------------- | --------|
| bind             | 只调用一次，指令第一次绑定到元素时调用。在此可以进行一次性的初始化设置 |
| inserted         | 被绑定元素插入父节点时调用(仅保证父节点存在，但不一定已被插入文档) |
| update           | 所在组件的VNode更新时调用，但是可能发生在其子VNode更新之前。指令的值可能发生了改变，也可能没有发生改变 |
| componentUpdated | 指令所在组件的VNode及其子VNode全部更新后调用  |
| unbind           | 只在指令与元素解绑时调用一次|

```html

<!DOCTYPE html>
<html>
 <head>
  <meta charset="utf-8">
  <title></title>
 </head>
 <body>
  <div id="app"  v-parameter:hello.a.b="message">
</div>
<script src="../../js/vue.js"" type="text/javascript" charset="utf-8"></script>
<script>
Vue.directive('parameter', {
  bind: function (el, binding, vnode) {
    var s = JSON.stringify
    el.innerHTML =
      'name: ' + s(binding.name) + '<br>' +
      'value: ' + s(binding.value) + '<br>' +
      'expression: ' + s(binding.expression) + '<br>' +
      'argument: ' + s(binding.arg) + '<br>' +
      'modifiers: ' + s(binding.modifiers) + '<br>' +
      'vnode keys: ' + Object.keys(vnode).join(', ')
  }
})
new Vue({
  el: '#app',
  data: {
    message: '前端学习!'
  }
})
</script>

 </body>
</html>
```

**效果**

![效果](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211115191243.png)

