---
title: 毕设-Fuzz-2
date: 2021-11-21 09:25:00
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

## Greybox Fuzzing

### Blackbox Mutation-base Fuzzer

在这个测试里面,似乎只要是在population里面的seed,在权重上是一样的,换句话说就是被挑选的概率是一样的.

不过,按照我的理解, 按说这个权重应该是要变化的,可能在后面的讲解中会讲到吧

### Greybox Mutation-base Fuzzer

我丢,从代码的角度上来看的话,灰盒测试无非是把`代码覆盖率`当成了加入`population`的一个准则

> 原话: If we reach new coverage,add inp to population and its coverage to population_coverage

### Boosted Greybox Fuzzer

果不其然, 这个增强版的就用到了`energy`,也就是上面所讲的权重,它用了一个函数来计算.

![计算权重的公式](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211121205144.png)

这个指数后面文章所取的值是5

❓我不太清楚是, `coverage`只是一个数字, 如果只是数字的话, 那如何衡量路径呢? 因为即使是路径不同, `coveraige`也有可能是一样的.

> emm,很明显,`coverage`应该不仅仅是数字,他应该是`(执行函数名,行数)`这样的结构组成的😨

![Boosted](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211121210348.png)

![Original](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211121210407.png)

把`energy`分配给哪些执行次数相对较少的路径上, 以期再获得其他路径.

> The exponential power schedule shaves some of the executions of the "high-frequency path" off and adds them to the lower-frequency paths. The path executed least often is either not at all exercised using the traditional power schedule or it is exercised much less often.

***Summary***. By fuzzing seeds more often that exercise low-frequency paths, we can explore program paths in a much more efficient manner.



------
看到现在的话,其实作者的思路,我们大概也清楚的知道了一些

- 1. 我们只是简单地随机产生字符串
- 2. 紧接着, 我们不满足于仅仅产生随机的字符串, 进而使用了`coverage`---`measure the effectiveness of different test generation techniques, but also to guide test generation towards code coverage.`
- 3. 有了`coverage`之后, 还不行, 因为总有一些路径几乎不被执行, 相反一些路径被执行的次数却很多. 倒也不是说这样不好, 只是 `By fuzzing seeds more often that exercise low-frequency paths, we can explore program paths in a much more efficient manner.` 所以又引入了 `Power Schedules`
- 4. 但是有了这些还是不行, 因为虽然产生的随机字符串有了一定的质量, 但是在语法上, 还是比较欠缺. 
    ![针对HtmlParser产生的字符串](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211121212317.png)

    > The greybox fuzzer executes much more complicated inputs, many of which include special characters such as opening and closing brackets and chevrons (i.e., <, >, [, ]). Yet, many important keywords, such as <html> are still missing.

    所以,下面一章就要将`grammars`的部分了

-----