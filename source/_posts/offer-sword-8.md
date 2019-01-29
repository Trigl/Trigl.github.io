---
layout:     post
title:      "「剑指 Offer」面试题 8：旋转数组的最小数字"
date:       2019-01-01 02:00:00
author:     "Ink Bai"
header-img: "/img/post/offer-sword-8.jpg"
tags:
    - 面试
---

> 把一个数组最开始的若干个元素搬到数组的末尾，我们称之为数组的旋转。输入一个非减排序的数组的一个旋转，输出旋转数组的最小元素。 例如数组 {3,4,5,1,2} 为 {1,2,3,4,5} 的一个旋转，该数组的最小值为 1。 NOTE：给出的所有元素都大于 0，若数组大小为 0，请返回 0。

首先分析一下题目，给定的是一个旋转数组，旋转数组是什么概念呢？就是原始数组是一个非减排序的数组，看到非减排序我们很容易想到使用二分查找的方法来实现，这里只不过是一个变形，那就是对数组旋转了一下。数组本来是非减排序的，那么旋转以后会变成怎样的呢？

普通情况下数组 {1,2,3,4,5} 旋转后变成 {3,4,5,1,2}，旋转后变成了两个非减部分，分别是 {3, 4, 5} 和 {1, 2}，并且左边整体比右边大。
特殊情况是整个数组 {1,2,3,4,5} 全部旋转，旋转后还是 {1,2,3,4,5}。

二分查找来分析的话，我们定义最小下标 low，最大下标 high，中间下标 mid = low + (high - low) / 2，然后开始循环，循环条件就是 low < high。

如果 array[mid] > array[high]，例如 {1,2,3,4,5} -> {3,4,5,1,2}，由于左边部分整体比右边大，我们可以判定最小值应该位于右边，所以此时设置 low = mid + 1。

如果 array[mid] < array[high]，例如 {1,2,3,4,5} -> {4,5,1,2,3}，此时最小值可能就位于 mid 下面或者位于 mid 的左边，因此设置 high = mid。

还有一种特殊情况是 array[mid] == array[high]，如 {0,1,1,1,1} 可以旋转为 {1,0,1,1,1}，此时最小值 0 位于左半部分；也可以旋转为 {1,1,1,0,1}，此时最小值 0 位于右半部分，所以这种情况是不确定的，需要一个值一个值的遍历，此时设置 high = high - 1。

通过二分查找解决的时间复杂度是 O(logN)，代码如下：

```java
import java.util.ArrayList;
public class Solution {
    public int minNumberInRotateArray(int [] array) {
        int low = 0;
        int high = array.length - 1;
        while(low < high) {
            int mid = low + (high - low) / 2;
            if(array[mid] > array[high]) {
                // 如果中间值大于最后一位的值，说明最小值在右半部分
                low = mid + 1;
            } else if (array[mid] == array[high]) {
                // 如果相等，不好确定最小值在左还是右，继续遍历
                high = high - 1;
            } else {
                // 如果中间值小于最后一位的值，说明最小值就是中间值或者位于左半部分
                high = mid;
            }
        }
        return array[low];
    }
}
```
