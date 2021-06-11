---
title: py如何根据字符串来创建对应的类
date: 2021-06-11 00:00:00
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
comments: true
# 文章标签 #
tags: python
# 文章分类 #
categories: 反射
# 文章摘要 #
description: 根据字符串来创建对应的类
# 文章置顶 param = 10000 #
sticky: 
---

# py如何根据字符串来创建对应的类

## 原理

py的反射原理，简单来说，反射就是能实现动态地调用方法\实例化对象。

举个例子:<br>

创建一个学生类Student的对象 person1、创建一个老师类Teacher的对象person1

```py
person1 = Student(name="张三")

or

person1 = Teacher(name="张三")
```

试想一下，假如，你并不是先前(在写程序之前)就知道这个person1的身份到底是学生还是老师，那你该如何创建这个对象？

或者说你要 **根据这个人的输入: 职业:老师,姓名:张三** 来动态的创建对象。

这里就要用到py的反射

对应到web路由可能更容易理解。详细请看 https://www.liujiangblog.com/course/python/48

## 需求

在本次实训的过程中，由于是基线检查，但是对于每个审查条目的规则(存储在数据库)是不一样的。

关键的是，所用的validator中预制的规则rule无法满足特定的需求。然后，除了使用他文档中的规则意外，我根据他自定义规则的写法，自定义如下三种规则

- AuditRule-判断前后集合是否一致
- AuditRuleInclude-判断前面集合是否是后面集合的子集
- AuditRuleSame-判断两个字符串是否相等。

```python
class AuditRule(Rule):
    """
    TestCode:
        rules = {"age": AuditRule('test,test')}
        req = {"age": 'test,test'}
        print(validate(req, rules,return_info=True))
    """

    def __init__(self, string: str):
        Rule.__init__(self)
        self.string = string
        self.value = string.split(',')

    def check(self, arg: str):
        if arg is None:
            arg = "Null"
            self.set_error("excepted get |" + self.string + "| but get |" + arg.replace("\"", "") + "|")
            return False
        exit_value_list = arg.split(',')
        # 判断两个集合是否一样  前面是否是后面的子集
        self.set_error("excepted get |" + self.string + "| but get |" + arg.replace("\"", "") + "|")
        return set(exit_value_list) == set(self.value)
```

其实，我所用的validator这个包，就已经利用了反射。因为他就是根据我输入的字符串，去动态地翻译和创建成对应的类。所以我也想实现根据数据库中存储的规则，来动态地创建。

![20210611161640](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20210611161640.png)


## 实现方式

### 使用的函数

我就只用到了**getattr**函数。其对应的文档解释如下:

```python
def getattr(object, name, default=None): # known special case of getattr
    """
    getattr(object, name[, default]) -> value
    
    Get a named attribute from an object; getattr(x, 'y') is equivalent to x.y.
    When a default argument is given, it is returned when the attribute doesn't
    exist; without it, an exception is raised in that case.
    """
    pass
```

### 实际操作

- 创建package rules 将自定义的三个类分别以.py的形式放进去
  
  ![20210611162135](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20210611162135.png)

- 在package中创建rules.py的文件，将自定义类，导入。**第一行不要也可以**

  ![20210611162237](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20210611162237.png)

- 在要使用的文件中，以**from rules import rules as Custom**的形式导入
- 编写相应的代码
  
  ```py
    for item in results:
    if item[3].startswith("Audit"):
        rules[item[0]] = getattr(Custom, item[3])(item[2])
    else:
        rules[item[0]] = item[1] + ":" + item[2]
  ```

### 总结

它对应的原理通过debug我猜测如下:
通过 `from rules import rules as Custom` 的方式其实是已经创建了``Custom``这个对象，其拥有三个自定义类的属性。

![20210611162753](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20210611162753.png)

然后通过``getattr``得到字符串对应的属性(类),并通过后面括号里面的字符串进行实例化。