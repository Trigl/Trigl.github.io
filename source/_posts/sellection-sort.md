---
layout:     post
title:      "选择排序"
date:       2019-01-29 02:00:00
author:     "Ink Bai"
catalog:    true
header-img: "/img/post/sellection-sort.jpg"
tags:
    - 算法与数据结构
---

## 思想
选择排序是最简单直观的一种算法，基本思想是每一趟从待排序的数据元素中选择最小（或最大）的一个元素作为首元素，直到所有元素排完为止。
听起来是不是跟冒泡排序很相似，实际上还是有差别的，冒泡排序是两两比较，然后两两交换，而选择排序是维护一个最小值（或最大值），每个数都和这个最小值比较，如果比这个最小值小，那么最小值更新，否则最小值不变。

## 算法步骤

1. 首先在未排序序列中找到最小（大）元素，存放到排序序列的起始位置
2. 再从剩余未排序元素中继续寻找最小（大）元素，然后放到已排序序列的末尾。
3. 重复第二步，直到所有元素均排序完毕。

## 代码实现

```java
import java.util.Arrays;

public class SelectionSort {
    public int[] sort(int[] sourceArray) {
        // 不改变原数组，在复制数组上操作
        int[] arr = Arrays.copyOf(sourceArray, sourceArray.length);

        // 做 arr.length - 1 次操作来实现完全有序
        for (int i = 0; i < arr.length - 1; i++) {
            int min = i;
            for (int j = i + 1; j < arr.length; j++) {
                if (arr[j] < arr[min]) {
                    // j 所在值比 min 所在值小，更新 min 的坐标
                    min = j;
                }
            }

            if (min != i) {
                // 如果 min 的值发生了改变，交换 min 和 i 的值
                swap(arr, min, i);
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
        SelectionSort s = new SelectionSort();
        int[] array = {3, 5, 3, 7, 14, 6, 11, 5};
        System.out.println(Arrays.toString(s.sort(array)));
    }
}
```

## 算法分析
时间复杂度：选择排序无论数组原始排序如何，每次都需要进行比较，因此时间复杂度一直都是 O(n^2)。
空间复杂度：不需要额外内存空间，空间复杂度 O(1)。
稳定性：举个例子，序列 5 8 5 2 9， 我们知道第一遍选择第 1 个元素 5 会和 2 交换，那么原序列中 2 个 5 的相对前后顺序就被破坏了；因此选择排序是不稳定的排序算法。
