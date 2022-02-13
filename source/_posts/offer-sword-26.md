---
layout:     post
title:      "「剑指 Offer」面试题 26：复杂链表的复制"
date:       2019-03-21 02:00:00
author:     "Ink Bai"
header-img: "/img/post/offer-sword-26.jpg"
catalog:    true
tags:
    剑指 Offer
---

> 输入一个复杂链表（每个结点中有结点值，以及两个指针，一个指向下一个结点，另一个特殊指针指向任意一个结点），返回结果为复制后复制链表的 head。

首先理解题意，这里说的复杂链表，说的是单链表，链表是一种比较简单的数据结构，但涉及到链表的操作却并不简单，原因就是因为链表里面有很多指针操作，一不小心可能就指错了或者是链表断了。

其实个人认为在做链表的操作只要弄清楚一点就容易很多，那就是 Java 传递参数的本质是什么，我们不用纠结 Java 是值传递还是引用传递，这两个概念很容易把人搞晕，只要知道对于基本数据类型，传参传的是这个值的副本，所以对参数的更改并不会影响其原始值；但是对于普通对象，传参传的是对象的引用的副本，由于这个副本和原本的引用指向的是同一个对象，所以对这个参数做更改是会改变对象的。

操作链表的时候我们不要纠结各个中间变量（参数），要站在原始对象的角度上去考虑问题。对于本题目来说，解题思路如下：

![](/img/content/copy-listnode.jpg)

具体代码如下：

```java
/*
public class RandomListNode {
    int label;
    RandomListNode next = null;
    RandomListNode random = null;

    RandomListNode(int label) {
        this.label = label;
    }
}
*/
public class Solution {
    public RandomListNode Clone(RandomListNode pHead) {
        if (pHead == null) return null;

        // 复制每个结点，将复制结点插入到原结点的后面，暂不考虑 random 结点
        RandomListNode node = pHead;
        while (node != null) {
            RandomListNode clone = new RandomListNode(node.label);
            RandomListNode next = node.next;
            clone.next = next;
            node.next = clone;
            node = next;
        }

        // 再重新遍历该链表，这次复制 random 结点
        node = pHead;
        while (node != null) {
            node.next.random = node.random == null ? null : node.random.next;
            node = node.next.next;
        }

        // 最后将整链表拆分开，得到原链表和复制链表
        node = pHead;
        RandomListNode cloneNode = pHead.next;
        while (node != null) {
            RandomListNode next = node.next.next;
            node.next.next = next == null ? null : next.next;
            node.next = next;
            node = node.next;
        }

        return cloneNode;
    }
}
```
