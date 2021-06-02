---
title: 实训day1-day2
date: 2021-06-1 00:00:00
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
description: git基本工具的使用+Spring入门1
# 文章置顶 #
sticky: 
---

# 实训day1-day2

目录:

- [实训day1-day2](#实训day1-day2)
  - [git工具的使用](#git工具的使用)
    - [关于git的原理](#关于git的原理)
    - [git 工作流](#git-工作流)
    - [为什么在```push```之前需要进行```pull```](#为什么在push之前需要进行pull)
    - [git处理冲突](#git处理冲突)
  - [spring项目入门](#spring项目入门)
    - [java反射](#java反射)
    - [思考](#思考)
      - [Spring IOC](#spring-ioc)
      - [Spring 依赖注入DI](#spring-依赖注入di)

## git工具的使用

### 关于git的原理

找到了一篇博客对于git的原理以及存储讲解的非常清楚

[https://zhaohuabing.com/post/2019-01-21-git/](https://zhaohuabing.com/post/2019-01-21-git/)

### git 工作流

你的本地仓库由 git 维护的三棵“树”组成。第一个是你的 ```工作目录```，它持有实际文件；第二个是 ```缓存区（Index）```，它像个缓存区域，临时保存你的改动；最后是 ```HEAD```，指向你最近一次提交后的结果。

![20210601113100](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20210601113100.png)

### 为什么在```push```之前需要进行```pull```

如果项目只有一个人，那无所谓。但是一般情况下，项目中都会有许多项目成员，在我们将自己的```分支 1``` 合并到 ```主分支 master```时，```主分支master```有可能已经发生改变(即成员2将自己的```分支2```合并到```主分支 master```之后```push```),此时如果直接```push```，会导致成员2所修改的部分被覆盖。

而在这之前进行```pull```操作，会把远程分支于本地分支进行合并。然后再进行```push```

git可能会在这种情况下，禁止你进行push操作

![20210601113613](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20210601113613.png)

### git处理冲突

git并不能智能化地解决不同开发者修改同一个文件的情况。如果不同开发者对同一文件进行了修改，那么这个冲突的过程，必须要手动解决，然后再次提交。

![20210601114236](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20210601114236.png)

![20210601114412](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20210601114412.png)

日志

![20210601114336](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20210601114336.png)

## spring项目入门

  基础的创建项目+运行web项目 没什么可以说的

### java反射

具体去看博客:https://www.cnblogs.com/chanshuyi/p/head_first_of_reflection.html

**反射就是在运行时才知道要操作的类是什么，并且可以在运行时获取类的完整构造，并调用对应的方法。**
一般情况下，我们使用某个类时必定知道它是什么类，是用来做什么的。于是我们直接对这个类进行实例化，之后使用这个类对象进行操作。

```java
Apple apple = new Apple(); //直接初始化，「正射」
apple.setPrice(4);
```

而反射则是一开始并不知道我要初始化的类对象是什么，自然也无法使用 new 关键字来创建对象了。
这时候，我们使用 JDK 提供的反射 API 进行反射调用：

```java
Class clz = Class.forName("com.chenshuyi.reflect.Apple");
Method method = clz.getMethod("setPrice", int.class);
Constructor constructor = clz.getConstructor();
Object object = constructor.newInstance();
method.invoke(object, 4);
```

上面两段代码的执行结果，其实是完全一样的。但是其思路完全不一样，第一段代码在未运行时就已经确定了要运行的类（Apple），而第二段代码则是在运行时通过字符串值才得知要运行的类（com.chenshuyi.reflect.Apple）

### 思考

为什么在浏览器中输入http://localhost:8080/index就能够访问到对应的IndexController中的index方法？

```java
@RestController
public class IndexController {

    @RequestMapping("/index")
    public String index(){
        return "Lakers win";
    }
}
```

简单来说，就是在运行时，浏览器通过获取```/index```找到了IndexController这个类（可能是Spring容器在启动之前或者之后创建好的），然后调用方法index，向前端返回 Lakers win

#### Spring IOC

详见博客：https://www.cnblogs.com/ysocean/p/7466217.html

IOC-Inversion of Control，即控制反转。它不是什么技术，而是一种设计思想。

&emsp;&emsp;传统的创建对象的方法是直接通过 new 关键字，而 spring 则是通过 IOC 容器来创建对象，也就是说我们将创建对象的控制权交给了 IOC 容器。我们可以用一句话来概括 IOC：

&emsp;&emsp;IOC 让程序员不在关注怎么去创建对象，而是关注与对象创建之后的操作，把对象的创建、初始化、销毁等工作交给spring容器来做。

项目加载时会扫描有注解```@RestController、@Controller、@Service、@Component```的类，通过反射创建这些类的对象放入Spring的容器 **(hashMap:key =》value -----indexController 名字 =》indexController的对象)**，需要使用的时候通过key直接取出来使用。

#### Spring 依赖注入DI

详见：http://c.biancheng.net/view/4253.html

依赖注入（Dependency Injection，DI）和控制反转含义相同，它们是从两个角度描述的同一个概念。

当某个 Java 实例需要另一个 Java 实例时，传统的方法是由调用者创建被调用者的实例（例如，使用 new 关键字获得被调用者实例），而使用 Spring 框架后，被调用者的实例不再由调用者创建，而是由 Spring 容器创建，这称为控制反转。

Spring 容器在创建被调用者的实例时，会自动将调用者需要的对象实例注入给调用者，这样，调用者通过 Spring 容器获得被调用者实例，这称为依赖注入。

依赖注入主要有两种实现方式，分别是属性 setter 注入和构造方法注入。
