---
layout:     post
title:      "「剑指 Offer」面试题 13：在 O(1) 时间删除链表结点"
date:       2019-01-08 03:00:00
author:     "Ink Bai"
header-img: "/img/post/offer-sword-13.jpg"
tags:
    剑指 Offer
---

> 给定单向链表的头指针和一个结点指针，定义一个函数在 O(1) 时间删除该结点。

在单向链表中删除一个结点，最常规的做法无疑是从链表的头结点开始，顺序遍历查找要删除的结点，并在链表中删除该结点，时间复杂度是 O(n)，显然不满足本题的要求。

之所以需要从头开始查找，是因为我们需要得到将被删除的结点前面一个结点。在单向链表中，结点中没有指向前一个结点的指针，所以只好从链表的头结点开始顺序查找。

那是不是一定需要得到被删除的结点的前一个结点呢？答案是否定的。我们可以很方便地得到要删除的结点的下一个结点。如果我们把下一个结点的内容复制到需要删除的结点上覆盖原有的内容，再把下一个结点删除，那是不是就相当于把当前需要删除的结点删除了？

不过上述思路还有一个问题：如果要删除的结点位于链表的尾部，那么它就没有下一个结点，怎么版？我们仍然从链表的头结点开始，顺序遍历得到该结点的前序结点，并完成删除操作。

最后需要注意的是，如果链表中只有一个结点，而我们又要删除链表的头结点（也是尾结点），此时我们在删除结点之后，还需要把链表的头结点设置位 null。

有了这些思路，可以得到下面的代码：

```java
public class Solution {
    class ListNode {
        int value;
        ListNode next;
    }

    public ListNode deleteNode(ListNode head, ListNode toBeDeleted) {
        if (head == null || toBeDeleted == null) return null;
        if (toBeDeleted.next != null) {
            // if toBeDeleted is not the last node
            toBeDeleted.value = toBeDeleted.next.value;
            toBeDeleted.next = toBeDeleted.next.next;
            return head;
        } else {
            if (head == toBeDeleted) {
                // if head node is the last node
                return null;
            } else {
                // traverse all nodes and delete the last
                ListNode node = head;
                while (node.next != toBeDeleted) {
                    node = node.next;
                }
                node.next = null;
                return head;
            }
        }
    }
}
```

接下来我们分析这种思路的时间复杂度。对于 n-1 个非尾结点而言，我们可以在 O(1) 时把下一个结点的内存复制覆盖要删除的结点，并删除下一个结点；对于尾结点而言，由于仍然需要顺序查找，时间复杂度是 O(n)。因此总的平均时间复杂度是 `[(n-1)*O(1)+O(n)]/n`，结果还是 O(1)。

值得注意的是，上述代码仍然不是完美的代码，因为它基于一个假设：要删除的结点的确在链表中。我们需要 O(n) 的时间才能判断链表中是否包含某一个结点。受到 O(1) 时间的限制，我们不得不把确保结点在链表中的责任推给了函数 deleteNode 的调用者。
