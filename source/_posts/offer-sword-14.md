---
layout:     post
title:      "「剑指 Offer」面试题 14：调整数组顺序使奇数位于偶数前面"
date:       2019-01-08 04:00:00
author:     "Ink Bai"
header-img: "/img/post/offer-sword-14.jpg"
tags:
    - 剑指 Offer
---

> 输入一个整数数组，实现一个函数来调整该数组中数字的顺序，使得所有奇数位于数组的前半部分，所有偶数位于数组的后半部分，并保证奇数和奇数，偶数和偶数之间的相对位置不变。

分析题目要求奇数在前偶数在后，并且重排序之后奇偶原来的顺序不能改变，这就说明我们只能顺序遍历所有的数字，遇到奇偶相邻就左右交换。具体做法就是在向后遍历的过程中用一个变量做为遍历过的奇数的数目，当遍历到的数字是奇数时，首先看一下前面有没有偶数，有的话就前移该奇数，这种方法的时间复杂度是 O(n^2)，具体代码如下：

```java
public class Solution {
    public void reOrderArray(int [] array) {
        if(array != null && array.length != 0) {
            // 标志奇数的数量
            int oddSize = 0;
            for(int i = 0; i < array.length; i++) {
                // 如果遍历到的数字是奇数
                if((array[i] & 1) == 1) {
                    int value = array[i];
                    // 奇数的前一位是否是偶数
                    int j = i - 1;
                    // 依次顺序交换该奇数前的偶数
                    while(j >= oddSize && (array[j] & 1) == 0) {
                        array[j + 1] = array[j];
                        array[j] = value;
                        j--;
                    }
                    oddSize++;
                }
            }
        }
    }
}
```
