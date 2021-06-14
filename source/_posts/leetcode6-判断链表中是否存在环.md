---
title: leetcode6-判断链表中是否存在环
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
description: 判断链表中是否存在环
# 文章置顶 param = 10000 #
sticky: 
---

# leetcode6-判断链表中是否存在环

这道题感觉很难嗷！ 但是确实是属于简单题的行列( 我是fw )，全程都在看解析。

## 思路

### 快慢指针

本方法需要读者对「Floyd 判圈算法」（又称龟兔赛跑算法）有所了解。

假想「乌龟」和「兔子」在链表上移动，「兔子」跑得快，「乌龟」跑得慢。当「乌龟」和「兔子」从链表上的同一个节点开始移动时，如果该链表中没有环，那么「兔子」将一直处于「乌龟」的前方；如果该链表中有环，那么「兔子」会先于「乌龟」进入环，并且一直在环内移动。等到「乌龟」进入环时，由于「兔子」的速度快，它一定会在某个时刻与乌龟相遇，即套了「乌龟」若干圈。

作者：LeetCode-Solution
链接：https://leetcode-cn.com/problems/linked-list-cycle/solution/huan-xing-lian-biao-by-leetcode-solution/
来源：力扣（LeetCode）

```c
bool hasCycle(struct ListNode* head) {
    if (head == NULL || head->next == NULL) {
        return false;
    }
    struct ListNode* slow = head;
    struct ListNode* fast = head->next;
    while (slow != fast) {
        if (fast == NULL || fast->next == NULL) {
            return false;
        }
        slow = slow->next;
        fast = fast->next->next;
    }
    return true;
}
```

#### 复杂度分析

- 时间复杂度：`O(N)`，其中 N 是链表中的节点数。
  
  - 当链表中不存在环时，快指针将先于慢指针到达链表尾部，链表中每个节点至多被访问两次。

  - 当链表中存在环时，每一轮移动后，快慢指针的距离将减小一。而初始距离为环的长度，因此至多移动 N 轮。

- 空间复杂度：`O(1)` 我们只使用了两个指针的额外空间。




### 哈希表

用哈希表来存储所有已经访问过的节点。每次我们到达一个节点，如果该节点已经存在于哈希表中，则说明该链表是环形链表，否则就将该节点加入哈希表中。重复这一过程，直到我们遍历完整个链表即可。

重要的是哈希表的原理
知乎的文章： 具体还是你后面去看看相应的源码，会比较方便一些。

https://zhuanlan.zhihu.com/p/144296454

#### 复杂度分析

- 时间复杂度：`O(N)`，其中 `N` 是链表中的节点数。最坏情况下我们需要遍历每个节点一次。

- 空间复杂度：`O(N)`，其中 `N` 是链表中的节点数。主要为哈希表的开销，最坏情况下我们需要将每个节点插入到哈希表中一次。
