---
title: leetcode4-删除链表节点
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
tags: leetcode
# 文章分类 #
categories: 链表
# 文章摘要 #
description: 删除链表节点
# 文章置顶 param = 10000 #
sticky: 
---

# leecode4-删除链表节点

## 题目描述

说明：文章中的优秀思路均来自优秀题解的第一个，之所以截图是因为懒。。
```C
给定单向链表的头指针和一个要删除的节点的值，定义一个函数删除该节点。

返回删除后的链表的头节点。

注意：此题对比原题有改动

示例 1:

输入: head = [4,5,1,9], val = 5
输出: [4,1,9]
解释: 给定你链表中值为 5 的第二个节点，那么在调用了你的函数之后，该链表应变为 4 -> 1 -> 9.
示例 2:

输入: head = [4,5,1,9], val = 1
输出: [4,5,9]
解释: 给定你链表中值为 1 的第三个节点，那么在调用了你的函数之后，该链表应变为 4 -> 5 -> 9.

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/shan-chu-lian-biao-de-jie-dian-lcof
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。
```

## 思路

这里借鉴了前面看到优秀思路中的**虚拟节点 virtualNode** ,即在头节点head前再增加一个虚拟节点，可以避免讨论 **tag** 节点是否是头节点的情况。最后统一返回 **virtualNode->next**

- 遍历链表找到值相等的节点
- 保留节点的前驱节点 **prev**
- 前驱节点 **prev** 指向删除节点 **tag** 的下一节点

### 代码

```C
struct ListNode* deleteNode(struct ListNode* head, int val) {
    struct ListNode* tag = head, * prev=NULL;
    struct ListNode* virtualNode = (ListNode*)malloc(sizeof(ListNode));
    virtualNode->next = head;
    virtualNode->val = -1;
    prev = virtualNode;
    while (tag->next)
    {
        if (tag->val == val) break;
        prev = tag;
        tag = tag->next;
    }
    prev->next = tag->next;
    return virtualNode->next;
}
```

## 优秀思路

### 递归

如果理解递归很困难，可以采用一种叫做**坚定信念**的理解方式。即假设**deleteNode返回的值就是对应节点的下一个节点**，那下面这个java版的递归就不难理解了。

![20210611093444](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20210611093444.png)
