---
layout:     post
title:      "「剑指 Offer」面试题 15：链表中倒数第 k 个结点"
date:       2019-01-08 05:00:00
author:     "Ink Bai"
header-img: "/img/post/offer-sword-15.jpg"
catalog:    true
tags:
    剑指 Offer
---

> 输入一个链表，输出该链表中倒数第 k 个结点。

本题目要求倒数第 k 个结点，也就是正数第 n-k+1 个结点，如果我们能够得到链表中结点的个数 n，那我们只要从头结点开始往后走 n-k+1 步就可以了。如何得到结点数 n？这个不难，只需要从头开始遍历链表，每经过一个结点，计数器加 1 就行了。

这种方法需要遍历链表两次，第一次统计出链表中结点的个数，第二次就能找到倒数第 k 个结点，那么有没有方法只遍历链表一次呢？

刚才我们定义了一个指针来顺序遍历，为了实现只遍历链表一次就能找到倒数第 k 个结点，我们可以定义两个指针。第一个指针从链表的头指针开始遍历向前走 k-1，第二个指针保持不懂；从第 k 步开始，第二个指针也开始从链表的头指针开始遍历。由于两个指针的距离保持在 k-1，当第一个（走在前面的）指针到达链表的尾结点是，第二个（走在后面的）指针正好是倒数第 k 个结点。

基本的思路考虑清楚以后让我们再考虑一下边界情况和异常情况：

- 如果该链表为空，那么也就不存在倒数第 k 个结点，此时应该也返回空。
- 如果 k 的值为 0，倒数第 0 个结点这种说法是无意义的，也应当返回空。
- 还有一种情况是输入的链表的结点总数少于 k，此时还是需要返回空。

综上，全面完善的代码如下：

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
    public ListNode FindKthToTail(ListNode head,int k) {
        if (head == null || k < 1) {
            return null;
        }
        // first index move to k then second index move
        ListNode ahead = head;
        ListNode behind = head;
        for (int i = 0; i < k - 1; i++) {
            if (ahead.next != null) {
                ahead = ahead.next;
            } else {
                // length of the node is less than k
                return null;
            }
        }
        // traverse to the end
        while (ahead.next != null) {
            ahead = ahead.next;
            behind = behind.next;
        }
        return behind;
    }
}
```

当我们用一个指针遍历链表不能解决问题的时候，可以尝试用两个指针来遍历链表。可以让其中一个指针遍历的速度快一些（比如一次在链表上走两步），或者让它先在链表上走若干步。
