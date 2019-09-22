---
layout:     post
title:      "「剑指 Offer」面试题 3：二维数组中的查找"
date:       2018-11-23 02:00:00
author:     "Ink Bai"
catalog:    true
header-img: "/img/post/offer-sword-3.jpg"
tags:
    剑指 Offer
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
    public static boolean Find(int target, int[][] array) {
        boolean found = false;

        if (array != null) {
            // 定义初始位置的下标，初始位置在右上角
            int row = 0;
            int column = array[0].length - 1;

            while (row <= array.length - 1 && column >= 0) {
                int value = array[row][column];
                if (target == value) {
                    // 目标值与当前值相同
                    return true;
                } else if (target > value) {
                    // 目标值比当前值大，下移，行增加
                    row++;
                } else {
                    // 目标值比当前值小，左移，列减小
                    column--;
                }
            }
        }

        return found;
    }

    // 测试代码
    public static void main(String[] args) {
        int[][] arr = {
                {1, 2, 8, 9},
                {2, 4, 9, 12},
                {4, 7, 10, 13},
                {6, 8, 11, 15}
        };

        // 1. 二维数组包含目标值
        // 查找的数字是数组中的最大值
        System.out.println(Find(15, arr));
        // 查找的数字是数组中的最小值
        System.out.println(Find(1, arr));
        // 查找的数字介于最大和最小值之间
        System.out.println(Find(7, arr));

        // 2. 二维数组不包含目标值
        // 查找的数字大于最大值
        System.out.println(Find(16, arr));
        // 查找的数字小于最小值
        System.out.println(Find(0, arr));
        // 查找的数字在最大最小值之间但是数组中没有
        System.out.println(Find(14, arr));

        // 3. 特殊输入：空指针
        System.out.println(Find(16, null));
    }
}
```

代码最坏情况是列递减到最小值，行递增到最大值，因此时间复杂度是 O(m+n)。
