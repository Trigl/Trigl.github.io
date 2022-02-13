---
layout:     post
title:      "「剑指 Offer」面试题 5：从尾到头打印链表"
date:       2018-11-27 02:00:00
author:     "Ink Bai"
header-img: "/img/post/offer-sword-5.jpg"
catalog:    true
tags:
    剑指 Offer
---

> 输入一个链表的头结点，从尾到头反过来打印出每个结点的值。

想要遍历一个链表，那么必然是从头开始依次遍历，现在要实现的是从尾到头，顺序反过来了，我们可以想一想有没有数据结构跟这种情况很类似呢？
从头到尾变成从尾到头，是不是一种典型的后进先出的形式？可以使用栈来轻松地解决这道题。

```java
/**
*    public class ListNode {
*        int val;
*        ListNode next = null;
*
*        ListNode(int val) {
*            this.val = val;
*        }
*    }
*
*/

import java.util.ArrayList;
import java.util.Stack;
public class Solution {
    public ArrayList<Integer> printListFromTailToHead(ListNode listNode) {
        Stack stack = new Stack<Integer>();
        // 首先从头到尾遍历链表，入栈
        while (listNode != null) {
            stack.push(listNode.val);
            listNode = listNode.next;
        }

        ArrayList arrayList = new ArrayList<Integer>();
        // 出栈，顺序变成了从尾到头
        while (!stack.isEmpty()) {
            arrayList.add(stack.pop());
        }
        return arrayList;
    }
}
```

这种解法的时间复杂度是 O(n)。
