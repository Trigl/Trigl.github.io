---
layout:     post
title:      "「剑指 Offer」面试题 16：反转链表"
date:       2019-01-09 01:00:00
author:     "Ink Bai"
header-img: "/img/post/offer-sword-16.jpg"
catalog:    true
tags:
    剑指 Offer
---

> 定义一个函数，输入一个链表的头结点，反转该链表并输出反转后链表的头结点。

又是一个链表问题，解决与链表相关的问题总是又大量的指针操作，本题目就是这样。需要反转链表，也就是本来指向下一个结点的指针要指到前一个，需要注意的是在这个过程中要特别注意保存好其中的某些结点，比如前一个结点或者下一个结点，这些是容易出错的点，代码如下：

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
    public ListNode ReverseList(ListNode head) {
        if(head == null) return null;

        // 反转后的链表表头
        ListNode reversedHead = null;
        ListNode next = null;

        while(head != null) {
            next = head.next;
            head.next = reversedHead;
            reversedHead = head;
            head = next;
        }

        return reversedHead;
    }
}
```
