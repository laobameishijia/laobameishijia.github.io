---
title: 实训day3
date: 2021-06-2 00:00:00
# 文章出处名称 #
from: 
# 文章出处链接 #
url: 
# 文章作者名称 #
author: 老叭美食家
# 文章作者签名 #
about: 
# 文章作者头像 #
avatar: 
# 是否开启评论 #
comments: false
# 文章标签 #
tags: 实训
# 文章分类 #
categories: 国信安实训
# 文章摘要 #
description: Spring入门2
# 文章置顶 #
sticky: 
---

- [Spring入门2](#spring入门2)
  - [MVC模式(mode\view\controller)](#mvc模式modeviewcontroller)
    - [MVC原理图](#mvc原理图)
  - [springMVC是什么](#springmvc是什么)
    - [原理图](#原理图)
  - [spring thymeleaf](#spring-thymeleaf)
    - [动静分离](#动静分离)
  - [Spring接口开发](#spring接口开发)
    - [类上的注解](#类上的注解)
    - [方法上的注解](#方法上的注解)
    - [参数注解](#参数注解)

# Spring入门2

## MVC模式(mode\view\controller)

详见博客 https://www.cnblogs.com/xiaoxi/p/6164383.html

### MVC原理图

![20210602172050](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20210602172050.png)

M-Model 模型（完成业务逻辑：有javaBean构成，service+dao+entity）

V-View 视图（做界面的展示  jsp，html……）

C-Controller 控制器（接收请求—>调用模型—>根据结果派发页面）

## springMVC是什么

　　springMVC是一个MVC的开源框架，springMVC=struts2+spring，springMVC就相当于是Struts2加上sring的整合，但是这里有一个疑惑就是，springMVC和spring是什么样的关系呢？这个在百度百科上有一个很好的解释：意思是说，springMVC是spring的一个后续产品，其实就是spring在原有基础上，又提供了web应用的MVC模块，可以简单的把springMVC理解为是spring的一个模块（类似AOP，IOC这样的模块），网络上经常会说springMVC和spring无缝集成，其实springMVC就是spring的一个子模块，所以根本不需要同spring进行整合。

### 原理图

![20210602172227](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20210602172227.png)

第一步:用户发起请求到前端控制器（DispatcherServlet）

第二步：前端控制器请求处理器映射器（HandlerMappering）去查找处理器（Handle）：通过xml配置或者注解进行查找

第三步：找到以后处理器映射器（HandlerMappering）像前端控制器返回执行链（HandlerExecutionChain）

第四步：前端控制器（DispatcherServlet）调用处理器适配器（HandlerAdapter）去执行处理器（Handler）

第五步：处理器适配器去执行Handler

第六步：Handler执行完给处理器适配器返回ModelAndView

第七步：处理器适配器向前端控制器返回ModelAndView

第八步：前端控制器请求视图解析器（ViewResolver）去进行视图解析

第九步：视图解析器像前端控制器返回View

第十步：前端控制器对视图进行渲染

第十一步：前端控制器向用户响应结果

## spring thymeleaf

详见 https://developer.aliyun.com/article/769977

模板引擎在web领域的主要作用：让网站实现界面和数据分离，这样大大提高了开发效率，让代码重用更加容易。

### 动静分离

对于传统jsp或者其他模板来说，没有一个模板引擎的后缀为.html，就拿jsp来说jsp的后缀为.jsp,它的本质就是将一个html文件修改后缀为.jsp，然后在这个文件中增加自己的语法、标签然后执行时候通过后台处理这个文件最终返回一个html页面。

浏览器无法直接识别.jsp文件，需要借助网络(服务端)才能进行访问；而Thymeleaf用html做模板可以直接在浏览器中打开。开发者充分考虑html页面特性，将Thymeleaf的语法通过html的标签属性来定义完成，这些标签属性不会影响html页面的完整性和显示。如果通过后台服务端访问页面服务端会寻找这些标签将服务端对应的数据替换到相应位置实现动态页面！大体区别可以参照下图

![20210602173047](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20210602173047.png)

上图的意思就是如果直接打开这个html那么浏览器会对th等标签忽视而显示原始的内容。如果通过服务端访问那么服务端将先寻找th标签将服务端储存的数据替换到对应位置。具体效果可以参照下图,下图即为一个动静结合的实例。

![20210602173204](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20210602173204.png)

## Spring接口开发

做如下区分的目的：方便后续代码的扩展和维护

### 类上的注解

Stererotype.Component标记Spring中普通组件
Stererotype.Controller 控制器
Stererotype.Service服务层对象、处理业务逻辑
Stererotype. Repository持久层对象、操作数据库
Web.bin.annotation.RestController web控制器，返回json数据

### 方法上的注解

@RequsetMapping 路由控制返回数据
@GetMapping get请求获取用户数据
@PostMapping 获取数据
@PutMapping 更新数据
@DeleteMapping 删除数据

### 参数注解

@RequestParam required参数是否必传、name别名(前端看到的)、defaultValue:默认值 (这个name很奇怪，不知道怎么用的)
