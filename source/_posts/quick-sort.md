---
layout:     post
title:      "快速排序"
date:       2019-01-29 06:00:00
author:     "Ink Bai"
catalog:    true
header-img: "/img/post/quick-sort.jpg"
tags:
    - 算法
---

## 思想
快速排序是由东尼·霍尔所发展的一种排序算法。在平均状况下，排序 n 个项目要 Ο(nlogn) 次比较。在最坏状况下则需要 Ο(n2) 次比较，但这种状况并不常见。事实上，快速排序通常明显比其他 Ο(nlogn) 算法更快，因为它的内部循环（inner loop）可以在大部分的架构上很有效率地被实现出来。
快速排序使用分治法（Divide and conquer）策略来把一个串行（list）分为两个子串行（sub-lists）。
快速排序又是一种分而治之思想在排序算法上的典型应用。本质上来看，快速排序应该算是在冒泡排序基础上的递归分治法。

## 算法步骤

1. 从数列中挑出一个元素，称为 “基准”（pivot）;
2. 重新排序数列，所有元素比基准值小的摆放在基准前面，所有元素比基准值大的摆在基准的后面（相同的数可以到任一边）。在这个分区退出之后，该基准就处于数列的中间位置。这个称为分区（partition）操作；
3. 递归地（recursive）把小于基准值元素的子数列和大于基准值元素的子数列排序；

## 代码实现
快排的代码实现有多种方式，有填坑法、交换法、顺序遍历法等，而且影响快排最大的是基准点的选取，因此出现了一些优化快排的算法例如三数取中法，这里我们用顺序遍历法来实现一下快排，图解如下：

![](/img/content/quick-sort.png)

```java
public class QuickSort {
    public int[] sort(int[] sourceArray) {
        // 不改变原数组，在复制数组上操作
        int[] arr = Arrays.copyOf(sourceArray, sourceArray.length);
        sort(arr, 0, arr.length - 1);
        return arr;
    }

    private void sort(int[] arr, int left, int right) {
        if (left < right) {
            // 找出中间点的位置并且将数据排在中间点两边
            int p = partition(arr, left, right);
            sort(arr, left, p - 1);
            sort(arr, p + 1, right);
        }
    }

    private int partition(int[] arr, int left, int right) {
        // 首先定义最右边的点为基准点，与基准点比较大小
        int pivot = right;
        int index = left;

        // 依次从左到右遍历每一个点，如果遍历到的点的值小于基准点的值，将该点与 index 交换，同时 index 加 1
        for (int i = left; i < right; i++) {
            if (arr[i] < arr[pivot]) {
                swap(arr, i, index);
                index++;
            }
        }

        // 这样遍历结束后 index 左边的点都比基准点的值小，右边的点都比基准点的值大，交换 pivot 和 index
        swap(arr, index, pivot);
        return index;
    }

    private void swap(int[] arr, int i, int j) {
        int temp = arr[i];
        arr[i] = arr[j];
        arr[j] = temp;
    }

    public static void main(String[] args) {
        QuickSort s = new QuickSort();
        int[] array = {3, 5, 3, 7, 14, 6, 11, 5};
        System.out.println(Arrays.toString(s.sort(array)));
    }
}
```

## 算法分析
时间复杂度：快速排序的最坏运行情况是 O(n²)，比如说顺序数列的快排。但它的平摊期望时间是 O(nlogn)，且 O(nlogn) 记号中隐含的常数因子很小，比复杂度稳定等于 O(nlogn) 的归并排序要小很多。所以，对绝大多数顺序性较弱的随机数列而言，快速排序总是优于归并排序。
空间复杂度：不需要额外空间，空间复杂度是 O(1)。
稳定性：在基准元素和 a[j] 交换的时候，很有可能把前面的元素的稳定性打乱，比如序列为 5 3 3 4 3 8 9 10 11， 现在基准元素 5 和 3 (第5个元素，下标从1开始计)交换就会把元素 3 的稳定性打乱，因此快排是不稳定排序。
