---
title: leetcode2-反转链表
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
description: 反转链表
# 文章置顶 param = 10000 #
sticky: 
---

# 反转链表

## 题目描述

定义一个函数，输入一个链表的头节点，反转该链表并输出反转后链表的头节点。

```c
示例:

输入: 1->2->3->4->5->NULL
输出: 5->4->3->2->1->NULL
 

限制：

0 <= 节点个数 <= 5000

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/fan-zhuan-lian-biao-lcof
```

## 思路

1. 链表没有节点
2. 链表只有一个节点
3. 链表有两个节点
4. 链表有三个及三个以上的节点

![用ipad把思路的图画在这](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/用ipad把思路的图画在这.png)

### 时间复杂度分析

由于只用遍历一遍链 <br>
时间复杂度为**O(n) n 为链表的长度** <br>
以上代码，**分配的空间不会随着处理数据量的变化而变化，因此得到空间复杂度为 O空间复杂度为O(1**)

## 优秀思路

这次优秀思路其实跟我思路差不多，但是优秀思路的代码写的要更简洁。

![20210608172423](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20210608172423.png)

## 我思路的代码

```C
struct ListNode* reverseList(struct ListNode* head) {
    ListNode* first,*second,*third;
    
    // 0个节点
    if (head == NULL) return NULL;
    // 1个节点
    if (head->next == NULL) return head;
    // 2个节点
    if (head->next->next == NULL) {
        first = head;
        second = head->next;
        first->next = NULL;
        second->next = first;
        return second;
    }
    // 3个以上的节点
    first = head;
    second = head->next;
    third = head->next->next;
    
    while (1) {
        second->next = first;
        if (third == NULL) break;

        first = second;
        second = third;
        third = third->next;
    }
    //把第一个节点的next指向null
    head->next = NULL;
    //返回头节点
    return second;
}
```


