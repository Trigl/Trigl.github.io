---
layout:     post
title:      "冒泡排序"
date:       2019-01-29 01:00:00
author:     "Ink Bai"
catalog:    true
header-img: "/img/post/bubble-sort.jpg"
tags:
    - 算法与数据结构
---

## 思想
冒泡排序（Bubble Sort）是一种简单直观的排序算法。它重复地走访过要排序的数列，一次比较两个元素，如果他们的顺序错误就把他们交换过来。走访数列的工作是重复地进行直到没有再需要交换，也就是说该数列已经排序完成。这个算法的名字由来是因为越小的元素会经由交换慢慢“浮”到数列的顶端。
作为最简单的排序算法之一，冒泡排序给我的感觉就像 Abandon 在单词书里出现的感觉一样，每次都在第一页第一位，所以最熟悉。冒泡排序还有一种优化算法，就是立一个 flag，当在一趟序列遍历中元素没有发生交换，则证明该序列已经有序。但这种改进对于提升性能来说并没有什么太大作用。

![](/img/content/bubble-sort.jpg)

## 算法步骤

1. 比较相邻的元素。如果第一个比第二个大，就交换他们两个。
2. 对每一对相邻元素作同样的工作，从开始第一对到结尾的最后一对。这步做完后，最后的元素会是最大的数。
3. 针对所有的元素重复以上的步骤，除了最后一个。
4. 持续每次对越来越少的元素重复上面的步骤，直到没有任何一对数字需要比较。

## 代码实现

```java
import java.util.Arrays;

public class BubbleSort {
    public int[] sort(int[] sourceArray) {
        // 不改变原数组，在复制数组上操作
        int[] arr = Arrays.copyOf(sourceArray, sourceArray.length);

        // 做 arr.length - 1 次操作来实现完全有序
        for(int i = 0; i < arr.length - 1; i++) {
            // 标记数组是否已经有序了，如果已经有序就不需要继续循环了
            boolean isSort = true;
            for(int j = 0; j < arr.length - 1 - i; j++) {
                if(arr[j] > arr[j + 1]) {
                    isSort = false;
                    // j 所在值比 j+1 所在值大，交换这两个位置的数
                    swap(arr, j, j + 1);
                }
            }

            // 如果在某次冒泡时发现 isSort 不是 false，说明没有交换操作，说明数组已经有序了，终止循环
            if(isSort) {
                break;
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
        BubbleSort s = new BubbleSort();
        int[] array = {3, 5, 3, 7, 14, 6, 11, 5};
        System.out.println(Arrays.toString(s.sort(array)));
    }
}
```

## 算法分析
时间复杂度：冒泡排序最好的情况是数组本身有序，此时只需 n-1 次比较即可，时间复杂度是 O(n)；最坏的情况是数组是逆序排序，此时需要比较 n-1+n-2+...+1=n(n-1)/2，交换次数和比较次数等值，所有时间复杂度是 O(n^2)；因此平均时间复杂度还是 O(n^2)。
空间复杂度：无需占用额外内存，空间复杂度是 O(1)。
稳定性：冒泡排序是数据两两比较，因此可以确定两个相等的值在排序前后其相对位置不会发生改变，是稳定的排序算法。
