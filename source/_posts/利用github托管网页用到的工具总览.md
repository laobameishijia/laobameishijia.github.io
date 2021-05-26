---
title: 利用github托管网页，用到的工具总结
date: 2021-05-26 00:00:00
# 文章出处名称 #
from: 
# 文章出处链接 #
url: 
# 文章作者名称 #
author:
# 文章作者签名 #
about: 
# 文章作者头像 #
avatar: 
# 是否开启评论 #
comments: false
# 文章标签 #
tags: 总结
# 文章分类 #
categories:待办
# 文章摘要 #
description: 利用github托管网页，用到的工具总结
# 文章置顶 #
sticky: 10000
---
## 待做
- [ ] 更改网页中js文件的cdn路径
- [ ] 分析原因Travis 中运行hexo deloy总是```remote: Invalid username or password.fatal: Authentication failed fo```
- [ ] 续费腾讯的对象存储cos，方便传输文件
- [ ] 将csdn上面的文件转过来

## 利用github托管网页，用到的工具总结

### Hexo(博客框架)
Hexo 是一个快速、简洁且高效的博客框架。Hexo 使用 Markdown（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。

### Travis CI(方便对博客更改，自动渲染)
Travis CI 提供的是持续集成服务（Continuous Integration，简称 CI）。它绑定 Github 上面的项目，只要有新的代码，就会自动抓取。然后，提供一个运行环境，执行测试，完成构建，还能部署到服务器。

持续集成指的是只要代码有变更，就自动运行构建和测试，反馈运行结果。确保符合预期以后，再将新代码"集成"到主干。

持续集成的好处在于，每次代码的小幅变更，就能看到运行结果，从而不断累积小的变更，而不是在开发周期结束时，一下子合并一大块代码。

### Valine - 一款快速、简洁且高效的无后端评论系统。
Valine 诞生于2017年8月7日，是一款基于LeanCloud的快速、简洁且高效的无后端评论系统。

理论上支持但不限于静态博客，目前已有Hexo、Jekyll、Typecho、Hugo、Ghost 等博客程序在使用Valine。

### LeanCloud （数据库)---评论、留言、文章数据统计
LeanCloud（原 AVOS Cloud） 是针对移动应用的一站式云端服务，专注于为应用开发者提供工具和平台。提供包括LeanStorage 数据存储、LeanMessage 通信服务、LeanAnalytics 统计分析、LeanModules 拓展模块等四大类型的后端云服务，加速应用开发。

### Algolia Search(数据库)--文章标签、分类统计
可以使用 GitHub 或者 Google 账户直接登录，注册后的 14 天内拥有所有功能（包括收费类别的）。之后若未续费会自动降级为免费账户，免费账户总共有 10,000 条记录，每月有 100,000 的可以操作数

### jsDelivr--js文件的cdn free
CDN的全称是Content Delivery Network，即内容分发网络。CDN是构建在网络之上的内容分发网络，依靠部署在各地的边缘服务器，通过中心平台的负载均衡、内容分发、调度等功能模块，使用户就近获取所需内容，降低网络拥塞，提高用户访问响应速度和命中率。CDN的关键技术主要有内容存储和分发技术。——百度百科