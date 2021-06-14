---
title: leetcode5-反向输出链表
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
description: 反向输出链表
# 文章置顶 param = 10000 #
sticky: 
---

# 反向输出链表

## 题目描述

```
输入一个链表的头节点，从尾到头反过来返回每个节点的值（用数组返回）。

 

示例 1：

输入：head = [1,3,2]
输出：[2,3,1]
 

限制：

0 <= 链表长度 <= 10000
```

## 思路

- 第一遍遍历找到一共的个数
- malloc
- 倒序赋值

### 代码

```C
//反序打印链表
int* reversePrint(struct ListNode* head, int* returnSize) {
    //第一遍遍历获取数目
    int num = 0;
    struct ListNode* temp = head;
    while (temp)
    {
        num++;
        temp = temp->next;

    }
    int* ret = (int*)malloc(num * sizeof(int));
    memset(ret, -1, num * sizeof(int));
    
    temp = head;
    int i = 1;
    while (temp)
    {
        ret[num - i] = temp->val;
        i++;
        temp = temp->next;
    }
    *returnSize = num;
    return ret;
}
```

### 复杂度分析

时间复杂度 O(n)
空间复杂度 O(n)

## 优秀思路

差不多跟我一样

```C
int* reversePrint(struct ListNode* head, int* returnSize){
    struct ListNode *p = head;
    int n = 0;
    while(p != NULL) {
        p = p->next;
        n++;
    }
    int *arr = (int *)malloc(sizeof(int) * n);
    struct ListNode *q = head;
    *returnSize = n;
    for(int i = n - 1; i >= 0; i--){
        arr[i] = q->val;
        q = q->next;
    }
    return arr;
}
```
