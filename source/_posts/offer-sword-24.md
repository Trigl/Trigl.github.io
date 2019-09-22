---
layout:     post
title:      "「剑指 Offer」面试题 24：二叉搜索树的后序遍历序列"
date:       2019-01-11 02:00:00
author:     "Ink Bai"
header-img: "/img/post/offer-sword-24.jpg"
tags:
    剑指 Offer
---

> 输入一个整数数组，判断该数组是不是某二叉搜索树的后序遍历的结果。如果是则返回 true，否则返回 false。假设输入的数组的任意两个数组都互不相同。

首先明确二叉搜索树的特点：左边结点小于根结点，右边结点大于根结点。

然后根据后序遍历我们可以确定最后一个结点是根结点，然后再循环找出左右结点的分界点，继续循环确保右边的数字都大于根结点，最后再递归查看左右两个子结点。

代码如下：

```java
public class Solution {
    public boolean VerifySquenceOfBST(int [] sequence) {
        if (sequence == null || sequence.length == 0) {
            return false;
        }
        return VerifySquenceOfBST(sequence, 0, sequence.length - 1);
    }

    private boolean VerifySquenceOfBST(int [] sequence, int start, int end) {
        if (start >= end) return true;
        int i = start;
        for (; i < end; i++) {
            if (sequence[i] > sequence[end]) break;
        }
        for (int j = i; j < end; j++) {
            if (sequence[j] < sequence[end]) return false;
        }

        return VerifySquenceOfBST(sequence, start, i - 1) && VerifySquenceOfBST(sequence, i, end - 1);
    }
}
```
