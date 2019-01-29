---
layout:     post
title:      "插入排序"
date:       2019-01-29 03:00:00
author:     "Ink Bai"
catalog:    true
header-img: "/img/post/insertion-sort.jpg"
tags:
    - 算法
---

## 思想
直接插入排序基本思想是每一步将一个待排序的记录，插入到前面已经排好序的有序序列中去，直到插完所有元素为止。

![](/img/content/insertion-sort.png)

## 算法步骤

1. 将第一待排序序列第一个元素看做一个有序序列，把第二个元素到最后一个元素当成是未排序序列。
2. 从头到尾依次扫描未排序序列，将扫描到的每个元素插入有序序列的适当位置。

## 代码实现

```java
import java.util.Arrays;

public class InsertionSort {
    public int[] sort(int[] sourceArray) {
        // 不改变原数组，在复制数组上操作
        int[] arr = Arrays.copyOf(sourceArray, sourceArray.length);

        // i = 0 视作有序部分，i > 1 视作无序部分，将无序部分的每一个数插入有序部分
        for (int i = 1; i < arr.length; i++) {
            int j = i;
            while (j > 0 && arr[j] < arr[j - 1]) {
                // 从有序数组的最大值开始比较，只要比之小就交换
                swap(arr, j, j - 1);
                j--;
            }
        }

        return arr;
    }

    private void swap(int[] array, int i, int j) {
        int temp = array[i];
        array[i] = array[j];
        array[j] = temp;
    }

    public static void main(String[] args) {
        InsertionSort s = new InsertionSort();
        int[] array = {3, 5, 3, 7, 14, 6, 11, 5};
        System.out.println(Arrays.toString(s.sort(array)));
    }
}
```

## 算法分析
时间复杂度：插入排序在最好的情况下，即有序的情况下，需要比较 n-1 次，时间复制度是 O(n)；在最坏的情况下时间复杂度是 O(n^2)；平均时间复杂度是 O(n^2)。
空间复杂度：O(1)。
稳定性：稳定排序。
