---
title: grafana iframe嵌入不显示的问题
date: 2021-05-26 00:00:00
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
tags: grafana
# 文章分类 #
categories: 配置
# 文章摘要 #
description: 
# 文章置顶 #
sticky: 
---
# grafana iframe嵌入不显示的问题

## 注意：
```grafana\grafana\conf```目录下有两个配置文件```defaults.ini```、```sample.ini```
- ```defaults.ini``` 这个才是grafana服务器真正运行时的配置文件
- ```sample.ini``` 只是个样例，别改错了

## 开启匿名登录
修改```grafana\grafana\conf```目录下的```defaults.ini```文件中的 ```[auth.anonymous]中的enabled = true```

![20210528084634](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20210528084634.png)

## 允许浏览器渲染iframe
修改上述文件中的```allow_embedding = true```

![20210528085103](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20210528085103.png)

# windows server重启grafana服务
由于grafana在运行之后已经被当作一个服务，可以在服务管理页面对其进行重启

![20210528085716](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20210528085716.png)

# 不显示的原因
grafana服务器响应头里面有一个```X-Frame-Options:deny```

![20210528085945](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20210528085945.png)

## X-Frame-Options
The X-Frame-Options HTTP 响应头是用来给浏览器 指示允许一个页面 可否在 ```<frame>, <iframe>, <embed> 或者 <object>``` 中展现的标记。站点可以通过确保网站没有被嵌入到别人的站点里面，从而避免 clickjacking 攻击。

有三个可能值
```
X-Frame-Options: deny
X-Frame-Options: sameorigin
X-Frame-Options: allow-from https://example.com/
```

如果设置为 deny，不光在别人的网站 frame 嵌入时会无法加载，在同域名页面中同样会无法加载。另一方面，如果设置为sameorigin，那么页面就可以在同域名页面的 frame 中嵌套。

- deny <br>
表示该页面不允许在 frame 中展示，即便是在相同域名的页面中嵌套也不允许。
- sameorigin <br>
表示该页面可以在相同域名页面的 frame 中展示。
- allow-from url <br>
表示该页面可以在指定来源的 frame 中展示。

## 修改之后，grafana服务器的响应头里不再包含这个字段
![20210528091108](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20210528091108.png)
就可以显示了
![20210528091142](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20210528091142.png)