---
title: Writeup for 0CTF2022
date: 2022-8-04 09:25:00
author: 美食家李老叭
img: https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220918152934.png
top: false
hide: false
cover: false
coverImg: 
password: 
toc: true
mathjax: false
summary: CTF
categories: CTF
tags:
    - 比赛
---
# Rev

## Baby Encoder----未完成

### 自己

#### 主逻辑

1. 利用 `\dev\urandom`随机生成了 `0x40` 的随机数字，并且对这些随机数字进行了线性变化。
   ![1](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220918153242.png)
2. 基于buf中八个字节，进行一系列运算，产生 `128`个double类型的数据到 `int_64 v6[1024]`中去。
   ![2部分函数](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220918153539.png)
3. 打印输出 `v6`中的内容，并且从标准输入中读取 `64个字节`，与 `buf前64个字节`进行对比，如果相同则输出flag。

#### 想法

通过观察发现 `\dev\urandom`每一次产生的随机数都是不一样的，所以我们只能是根据程序回显出来的v6中的内容反推buf。

在执行想法的过程中遇到了阻碍，在主逻辑的第二步中，参与运算的部分数据同样来源于 `\dev\urandom`。这样我就没办法通过 `v6的内容`根据运算过程推算出参与运算的 `int_64 buf[j]--8个字节的内容`。

```c
  v24 = (int)(3.141592653589793 / 64.0 * (double)((int)sub_55C6B1D5C365() % 255));
  v25 = (int)(3.141592653589793 / 64.0 * (double)((int)sub_55C6B1D5C365() % 255));
  v26 = (int)(3.141592653589793 / 64.0 * (double)((int)sub_55C6B1D5C365() % 255));
  v27 = (int)(3.141592653589793 / 64.0 * (double)((int)sub_55C6B1D5C365() % 255));
  v28 = (int)(3.141592653589793 / 64.0 * (double)((int)sub_55C6B1D5C365() % 255));
  v29 = (int)(3.141592653589793 / 64.0 * (double)((int)sub_55C6B1D5C365() % 255));
  v30 = (int)(3.141592653589793 / 64.0 * (double)((int)sub_55C6B1D5C365() % 255));
  v31 = (int)(3.141592653589793 / 64.0 * (double)((int)sub_55C6B1D5C365() % 255));

  result = (unsigned int)((int)sub_55C6B1D5C365() % 255);
  v32 = result;
  for ( i = 0; i <= 127; ++i )
  {
    v11 = cos((3.141592653589793 + 3.141592653589793) * 2000.0 * 0.00000390625 * (double)i + (double)v24) * (double)a2;
    v12 = cos(
            (2000.0 * (3.141592653589793 + 3.141592653589793) + 2000.0 * (3.141592653589793 + 3.141592653589793))
          * 0.00000390625
          * (double)i
          + (double)v25)
        * (double)a3
        + v11;
    v13 = cos((3.141592653589793 + 3.141592653589793) * 2000.0 * 3.0 * 0.00000390625 * (double)i + (double)v26)
        * (double)a4
        + v12;
    v14 = cos((3.141592653589793 + 3.141592653589793) * 2000.0 * 4.0 * 0.00000390625 * (double)i + (double)v27)
        * (double)a5
        + v13;
    v15 = cos((3.141592653589793 + 3.141592653589793) * 2000.0 * 5.0 * 0.00000390625 * (double)i + (double)v28)
        * (double)a6
        + v14;
    v16 = cos((3.141592653589793 + 3.141592653589793) * 2000.0 * 6.0 * 0.00000390625 * (double)i + (double)v29)
        * (double)a7
        + v15;
    v17 = cos((3.141592653589793 + 3.141592653589793) * 2000.0 * 7.0 * 0.00000390625 * (double)i + (double)v30)
        * (double)a8
        + v16;
    v18 = cos((3.141592653589793 + 3.141592653589793) * 2000.0 * 8.0 * 0.00000390625 * (double)i + (double)v31)
        * (double)a9
        + v17;
    v10 = v18 + (double)((int)sub_55C6B1D5C365() % 3);
    result = 8LL * i + a1;
    *(double *)result = (double)v32 + v10;
  }
```

然后我有猜测，是否在 `cos参与的运算`中可以把相关的随机因素给消除掉。观察了很长时间，也没发现其中蕴含的数学法则可以把随机的因素给消除掉。



### Writeup

居然涉及到数学的傅里叶级数，这真的是完全没有想到的。

大体思路就是通过傅里叶级数的不同形式，忽略掉随机的部分，只求振幅。振幅就是`buf[j]`。咱是实在没耐心看着一堆数学公式。

[writeup](https://hackmd.io/@vishiswoz/rkSNRg8Zo#)

![writeup图片](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220920151256.png)

## vintage - part1---未完成

### 自己

用linux下的file工具分析了一下，显示其是 `Vectrex ROM image`。

![Vectrex ROM](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220918161455.png)

我找了很长时间的模拟器，想要找一个模拟器模拟运行这个镜像文件。找了半天，找到了一个在线版本的。把这个二进制文件直接拖进去就可以运行。

[https://drsnuggles.github.io/jsvecx/](https://drsnuggles.github.io/jsvecx/)

![运行结果](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220918173202.png)

泥马8位密码，大写字母+数字+符号。呜呜呜，然后我试着在二进制文件中找了一下字符串，如果游戏密码是明文存放的话，应该是可以找到的。

结果就是没找着。。。。

我又想着根据这个game的文件格式---通过对比网上的这种游戏发现它们都是 `.vec结尾的`去逆推源码。在网上搜了半天也没发现相关。真是无语~~~

:> :> :> :>

最终找到了一个vectrexy工具链接如下，倒是可以调试了，但是这里面代码啥的看不懂呢！

[https://github.com/amaiorano/vectrexy](https://github.com/amaiorano/vectrexy)

![vectrexy](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220919100935.png)

### writeup

## 总结

😔😔😔😔