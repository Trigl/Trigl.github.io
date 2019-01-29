---
layout:     post
title:      "「剑指 Offer」面试题 3：二维数组中的查找"
date:       2018-11-23 02:00:00
author:     "Ink Bai"
catalog:    true
header-img: "/img/post/offer-sword-3.jpg"
tags:
    - 面试
---

> 题目：完成一个函数，输入一个二维数组和一个整数，判断数组中是否含有该整数。这个二维数组的结构是，每一行从左到右递增，每一列从上到下递增。

## 思路
数组的结构是从上到下递增从左到右递增，我们可以从右上角开始往左下角移动：

若当前值与目标值相等，说明这个值存在，over

![](/img/content/find-number-01.jpg)

若当前值比目标值大，由于从上到下递增，所以所在列的值都会比目标值大，左移一列

![](/img/content/find-number-02.jpg)

若当前值比目标值小，由于从左到右递增，所以所在行的值都比目标值小，下移一行

![](/img/content/find-number-03.jpg)

## 实现代码

```java
public class Solution {
    public boolean Find(int target, int [][] array) {
        // 以右上角的点为起始点
        int row = 0;
        int col = array[0].length - 1;

        while(row < array.length && col >= 0) {
            if(array[row][col] == target) {
                // 将遍历值与目标比较，相等返回 true
                return true;
            } else if(array[row][col] > target) {
                // 比目标值大，左移一列
                col--;
            } else {
                // 比目标值小，下移一行
                row++;
            }
        }
        return false;
    }
}
```
