---
title: CSAW-CTF--Write-Rev
date: 2022-9-10 09:25:00
author: 美食家李老叭
img: https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220910155740.png
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

## DockREleakage

这个题目就比较简单，感觉不太像是逆向，有点像是溯源。

![文件目录结构](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220910160143.png)

既然题目中说到了,隐私的数据要保管好。那么大概就是直接在文件中出现的。

打开acb.....这个文件之后，找到了flag的一部分，文件中说明了，剩下的flag要我们自己去找。所以呢，继续去找其他的文件。
![flag的一部分](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220910160356.png)

最终在另一个文件中找到了剩余的flag

![flag剩余的部分](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220910160527.png)

## Anya Gacha

### 关键

data = "wakuwaku" ---> byte类型 "77616b7577616b75"
sha256(byte(data)) ----> 产生的是256bit的数据
注意，由于sha256产生的256bit的数据，所以接下来的编码方式就很重要。
如果你把这256bit的数据 转换为 16进制的字符串，那么应该是64个字符。
这个时候，如果你还想继续进行hash运算，你又要将`16进制字符串转为byte类型`。而此时转换为的`byte类型是512bit`，因为是要按照`utf-8`的编码方式进行。

### 解析

这个题目，提供了一个根据unity写的游戏。下载对应系统版本之后打开，我下载的是win版本的。

游戏页面中说明了，保证在1000内抽到这个人物。而这个人物会告诉你答案。有点类似于某些游戏的抽奖保底机制。

![游戏页面](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220910160725.png)

根据网上的资料，要想逆向`unity`写的游戏，`dnsPy`这个工具不可或缺。下载完成之后，把包含程序主题逻辑的`Assembly-CSharp.dll`文件送入。

![Assembly-CSharp.dll](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220910161029.png)

反汇编之后的代码都是明文的，C#语言读起来也非常友好。很容易我们就发现了关键函数。

![wish函数](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220910161219.png)
![upload函数](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220910161248.png)

阅读之后程序的主题逻辑也就清楚了

- 每点击一次wish--抽奖，程序会把初始字符串`wakuwaku`进行一次hash运算
- 然后通过base64加密之后将其发往固定的服务器
- 判断服务器是否返回数据---经过测试，如果是不正确的hash，目标服务器不会返回任何数据。

所以，我们就直接hash 1000次，然后base64加密之后，发给服务器就行。工具我用的是filder

```py
import hashlib
import base64

data = "wakuwaku"   # 要进行加密的数据
data = data.encode('utf-8')
for x in range(0,1000):
    data_sha = hashlib.sha256(data).digest()
    data = data_sha

b64_byt = base64.b64encode(data)
print(b64_byt )
```

## game

### 关键

这里qmemcpy函数的第二个参数表面上看上去只有4个字节，但实际上传递到该函数中的只是`指向第一个字符的指针`。\

![memcopy](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220911165015.png)

之所以在反汇编函数中仅仅出现cook这四个字母，是因为cook后面保存的是00。让编译器误认为其字符串结束了。以后在遇到的时候，就直接当指针处理。

![20220911165307](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220911165307.png)

### 解析

本题提供了game 的exe程序，但真正的game程序部署在服务器上。我们需要根据现有的程序，推断程序的逻辑。再通过nc，得到五个被切分的flag文件，最后提交。

程序大体逻辑为，其通过设置迷宫游戏中五个特殊的位置。当走到这五个特殊的位置时，会提示你输入密码。如果你输入的密码正确，会显示flag的一部分。当你把全部的特殊位置全部解决之后，就可以按顺序把正确的flag拼接出来。

通过下面这个函数，我们可以看得出来`v7`的值只能是`0,1,2,3,4`。那么对应的`v8`同样也只能是 `0,1,2,3,4`

![关键函数1](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220911165425.png)

然后是`fnv_1a_32`函数,这里我不明白为什么反汇编出来居然有三个参数。但是通过汇编代码来看的话，只有一个参数。该函数就是异或操作，我们需要定位的就是这个参数。而这个参数通过`v8`就能确定。

![fnv_1a_32汇编代码](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220911170324.png)
![函数逻辑](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220911170633.png)

因为v11已知，所以我们根据`v8`可以计算出相应的`pass`。

```py

while_count = [0, 1, 2, 3, 4]
a = [
    0x63, 0x6f, 0x6f, 0x6b,
    0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x66, 0x6c,
    0x61, 0x77, 0x65, 0x64,
    0x00, 0x00, 0x00, 0x00,
    0x67, 0x72, 0x61, 0x76,
	0x65, 0x6c, 0x00, 0x00,
    0x00, 0x00, 0x6b, 0x69,
    0x6e, 0x67, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00,
    0x64, 0x65, 0x63, 0x69,
    0x73, 0x69, 0x76, 0x65,
    0x00, 0x00
]

for k in while_count:
    result = 2166136261
    part = a[10*k:]
    #print(str(part))
    for i in part:
        if i == 0:
            break
        result = 16777619*(i ^ (result & 0xFFFFFFFF))
    print(result)

```

这个函数可以计算出五个不同的值，接下来要做的，就是通过nc链接服务器，找到迷宫中的特殊位置，尝试这五个不同的密码。拼接所有的flag。

![结果](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220911171154.png)

这里找到特殊位置的方法，不知道有没有什么窍门。反正我是一个一个试的，纯粹是按照遍历的方法试出来的。按照顺序试就行。

![1-1路径](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220911171521.png)

最终flag为`flag{e@5+er_e995_6ehind_p@yw@115_i5_+he_dum6e5+_ide@_ever!!}`

## 