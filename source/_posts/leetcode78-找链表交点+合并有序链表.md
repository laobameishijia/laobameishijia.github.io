---
title: leetcode7/8-找链表交点/合并有序链表
date: 2021-06-8 00:00:00
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
description: 找链表交点/合并有序链表
# 文章置顶 param = 10000 #
sticky: 
---

- [leetcode7-找链表交点](#leetcode7-找链表交点)
  - [题目描述](#题目描述)
  - [思路](#思路)
    - [代码](#代码)
    - [复杂度分析](#复杂度分析)
- [leetcode8-合并有序链表](#leetcode8-合并有序链表)
  - [题目描述](#题目描述-1)
  - [思路](#思路-1)
    - [代码](#代码-1)
    - [复杂度分析](#复杂度分析-1)


# leetcode7-找链表交点

这个题目又没有好好审题，我以为的交点可以是这样的

![20210615113259](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20210615113259.png)

没想到交点以后的所有节点应该都是重合的！

![20210615113350](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20210615113350.png)

## 题目描述


## 思路

如果有相交的结点D的话,每条链的头结点先走完自己的链表长度,然后回头走另外的一条链表,那么两结点一定为相交于D点,因为这时每个头结点走的距离是一样的,都是 AD + BD + DC,而他们每次又都是前进1,所以距离相同,速度又相同,固然一定会在相同的时间走到相同的结点上,即D点。

- 如果不相交 ： 如果不相交的话 假设两个链表长度不相等 一个为A 一个为B ，指针第一次走完A会去走B,另一个走完B再去走A，两个指针走的路程都是A+B。会同时为NULL 跳出循环

- 如果不相交且链表长度相等: 那么一个指针走A,一个指针走B，它俩同时走到NULL，相等，跳出循环

### 代码

```C
struct ListNode *getIntersectionNode(struct ListNode *headA, struct ListNode *headB) {
    struct ListNode* A, * B;
    A = headA;
    B = headB;
    while(A!=B){
        A = A == NULL ? headB : A->next;
        B = B == NULL ? headA : B->next;
    }
    return A;
}
```
### 复杂度分析

时间复杂度 `O(N)` 最差依次访问一遍 `A+B` 中的所有节点 <br>
空间复杂度 `O(1)` 就用两个指针

# leetcode8-合并有序链表

## 题目描述

```C
输入两个递增排序的链表，合并这两个链表并使新链表中的节点仍然是递增排序的。

示例1：

输入：1->2->4, 1->3->4
输出：1->1->2->3->4->4

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/he-bing-liang-ge-pai-xu-de-lian-biao-lcof
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。
```

## 思路

leetcode的题解

![20210615164632](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20210615164632.png)

### 代码

```C
struct ListNode* mergeTwoLists(struct ListNode* l1, struct ListNode* l2){
    struct ListNode* a, *b,*c,*d;
    a = l1;
    b = l2;
    d = c = (struct ListNode*)malloc(sizeof(struct ListNode));
    while(a&&b){
        if(a->val < b->val) {
            c->next = a;
            a = a->next;
            c = c->next;
        }
        else if(a->val>=b->val){
            c->next = b;
            b = b->next;
            c = c->next;
        }
    }
    if(a == NULL) c->next = b;
    else c->next = a;
    return d->next;
}
```

### 复杂度分析

时间复杂度 `O(M+N)` M为l1链表的长度 N为l2链表的长度 <br>
空间复杂度 O(1)
