---
title: leetcode-1-单链表输出倒数第k个节点
date: 2021-06-6 00:00:00
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
tags: leetcode
# 文章分类 #
categories: 链表
# 文章摘要 #
description: 单链表输出倒数第k个节点
# 文章置顶 #
sticky: 
---

# 单链表输出倒数第k个节点

## 思路

- 遍历得到链表的节点个数
- 再根据节点个数和k得到目标节点的正向序号
- 遍历链表找到该节点

### 代码

#### 单链表版

```C

struct ListNode {
    int val;
    struct ListNode* next;
};

struct ListNode* getKthFromEnd(struct ListNode* head, int k) {
    int all = 0;
    ListNode* temp = head;
    while (temp->next)
    {
        all++;
        temp = temp->next;
    }
    all = all + 1;//加上最后一个节点

    int num = all - k + 1;
    if (num < 1) return NULL;
    else
    {
        temp = head;
        while (num != 1)
        {
            temp = temp->next;
            num--;
        }
        return temp;
    }
}

int main()
{
    ListNode *ahead, *after, *head, *result, *temp;
    ahead = (struct ListNode*)malloc(sizeof(ListNode));
    ahead->val = 1;
    head = ahead;
    for (int i = 1; i < 7; i=i+1) {
        after = (struct ListNode*)malloc(sizeof(ListNode));
        after->val = i + 1;
        after->next = NULL;

        ahead->next = after;
        ahead = after;
    }
    temp = head;
    while (1) {
        printf_s("%d->", temp->val);
        if (temp->next==NULL) {
            printf_s("\n%s", "跳出循环");
            break;
        }
        temp = temp->next;
    }
    
    result = getKthFromEnd(head, 1);
    if (result->next) {
        printf_s("\n%d->%d", result->val, result->next->val);//这里有可能result没有next节点
    }
    else 
        printf_s("\n%d", result->val);


}
```

#### 双链表版

```C
//双向链表版
struct ListNode {
    int val;
    struct ListNode* next;//前向指针
    struct ListNode* previous;//后向指针
};

struct ListNode* getKthFromEnd(struct ListNode* head, int k) {

    ListNode* temp = head;
    while (temp->next)
    {
        temp = temp->next;
    }

    if (k < 1) return NULL;
    else
    {
        while (k != 1)
        {
            temp = temp->previous;
            k--;
        }
        return temp;
    }
}

int main()
{
    ListNode* ahead, * after, * head, * result, * temp;
    ahead = (struct ListNode*)malloc(sizeof(ListNode));
    ahead->val = 1;
    ahead->previous = NULL;

    head = ahead;
    //temp = ahead;
    for (int i = 1; i < 7; i = i + 1) {
        after = (struct ListNode*)malloc(sizeof(ListNode));
        after->val = i + 1;
        after->next = NULL;
        after->previous = ahead;
        
        ahead->next = after;
        ahead = after;
    }
    temp = head;
    while (1) {
        printf_s("%d->", temp->val);
        if (temp->next == NULL) {
            printf_s("\n%s", "跳出循环");
            break;
        }
        temp = temp->next;
    }

    result = getKthFromEnd(head, 2);
    if (result->next) {
        printf_s("\n%d->%d", result->val, result->next->val);//这里有可能result没有next节点
    }
    else
        printf_s("\n%d", result->val);

}
```

## 优秀解题思路

- 初始化两个指针a,b 指向头节点
- b指针先往前走k个节点
- a,b指针同时向前走，直到b为空指针

### 代码

```C
struct ListNode* getKthFromEnd(struct ListNode* head, int k){
    struct ListNode *prev, *cur;
    prev = head;
    cur = head;
    for(k=k-1;k>0;k--){
        cur = cur->next;
    }
    while(cur->next != NULL){
        prev = prev->next;
        cur = cur->next;
    }
    return prev;
}
```