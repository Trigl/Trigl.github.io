---
layout:     post
title:      "「剑指 Offer」面试题 17：合并两个排序的链表"
date:       2019-01-09 02:00:00
author:     "Ink Bai"
header-img: "/img/post/offer-sword-17.jpg"
catalog:    true
tags:
    剑指 Offer
---

> 输入两个递增排序的链表，合并这两个链表并使新链表中的结点仍然是按照递增排序的。

首先分析一下合并的过程，比较两个链表的头结点的值，最后的结果要递增排序，因此比较小的值应当是合并以后新链表的头结点，单独拎出来，剩下的两个链表继续进行合并。从这个过程可以看出来，这是一个典型可以用递归解决的问题，总结起来就是首先确定头结点，然后递归合并剩下的两个链表即可。

接下来我们考虑异常情况。每当代码试图访问空指针指向的内存时程序就会崩溃，从而导致鲁棒性问题。在本题中一旦输入空的链表就会引入空的指针，因此我们要对空链表单独处理。当第一个链表是空链表，也就是它的头结点是一个空指针时，那么把它和第二个链表合并，显然合并的结果就是第二个链表。同样，当输入的第二个链表的头结点是空指针时，我们把它和第一个链表合并得到的结果就是第一个链表。如果两个链表都是空链表，合并的结果是得到一个空链表。

想清楚合并的过程，下面就是具体的代码：

```java
/*
public class ListNode {
    int val;
    ListNode next = null;

    ListNode(int val) {
        this.val = val;
    }
}*/
public class Solution {
    public ListNode Merge(ListNode list1,ListNode list2) {
        if (list1 == null) {
            return list2;
        }
        if (list2 == null) {
            return list1;
        }

        ListNode mergedList = null;
        if (list1.val < list2.val) {
            mergedList = list1;
            mergedList.next = Merge(list1.next, list2);
        } else {
            mergedList = list2;
            mergedList.next = Merge(list1, list2.next);
        }
        return mergedList;
    }
}
```
