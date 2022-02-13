---
layout:     post
title:      "「剑指 Offer」面试题 9：斐波那契数列"
date:       2019-01-02 02:00:00
author:     "Ink Bai"
header-img: "/img/post/offer-sword-9.jpg"
catalog:    true
tags:
    剑指 Offer
---
> 输入一个整数 n，输出斐波那契数列的第 n 项。斐波那契数列的定义如下：
F0 = 0
F1 = 1
Fn = F[n-1]+ F[n-2] (n > 1)

这个题目看到的第一眼，很容易就想到使用递归的方法来做，只要我们知道了斐波那契数列第 0 项和第 1 项的值，其他项的值都可以通过这两项递归累加得到。

但是这样真的没问题吗？每次使用递归的时候我们都要考虑一下有没有性能问题，比如求解 F(10) 的时候，需要先得到 F(9) 和 F(8)。要得到 F(9)，需要先得到 F(8) 和 F(7)，我们可以用下面的树形结构来表示这种以来关系：

![](/img/content/Fibonacci.jpg)

可以看到这棵树中很多结点都是重复的，而且重复的结点数会随着 n 的增大而急剧增加，时间复杂度是以 n 的指数倍递增的，因此递归的方式是不可行的。

那么我们如何改进一下呢？上面递归的方法慢就是因为我们从上往下计算，有太多结点需要重复计算。更简单的方法是从下往上计算，首先根据 F(0) 和 F(1) 得到 F(2)，然后把 F(1) 和 F(2) 作为中间结果保存起来，然后计算出 F(3)，以此类推就可以算出第 n 项了，这种方式的时间复杂度是 O(n)，实现代码如下：

```java
public class Solution {
    public int Fibonacci(int n) {
        int fibNMinusOne = 1;
        int fibNMinusTwo = 0;
        int fibN = 0;
        for(int i = 2; i <= n; i++) {
            fibN = fibNMinusOne + fibNMinusTwo;
            fibNMinusTwo = fibNMinusOne;
            fibNMinusOne = fibN;
        }
        if(n == 0) {
            return fibNMinusTwo;
        } else if (n == 1) {
            return fibNMinusOne;
        } else {
            return fibN;
        }
    }
}
```
