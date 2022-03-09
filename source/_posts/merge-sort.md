---
layout:     post
title:      "归并排序"
date:       2019-01-29 05:00:00
author:     "Ink Bai"
catalog:    true
header-img: "/img/post/merge-sort.jpg"
tags:
    - 算法与数据结构
---

## 思想
归并排序（MERGE-SORT）是利用归并的思想实现的排序方法，该算法采用经典的分治（divide-and-conquer）策略（分治法将问题分(divide)成一些小的问题然后递归求解，而治(conquer)的阶段则将分的阶段得到的各答案"修补"在一起，即分而治之)。

![](/img/content/merge-sort1.jpg)

可以看到这种结构很像一棵完全二叉树，本文的归并排序我们采用自上而下的递归，也可以采用自下而上的迭代实现。分阶段可以理解为就是递归拆分子序列的过程，递归深度为log2n。

再来看看治阶段，我们需要将两个已经有序的子序列合并成一个有序序列，比如上图中的最后一次合并，要将[4,5,7,8]和[1,2,3,6]两个已经有序的子序列，合并为最终序列[1,2,3,4,5,6,7,8]，来看下实现步骤。

![](/img/content/merge-sort2.jpg)

## 算法步骤

1. 申请空间，使其大小为两个已经排序序列之和，该空间用来存放合并后的序列；
2. 设定两个指针，最初位置分别为两个已经排序序列的起始位置；
3. 比较两个指针所指向的元素，选择相对小的元素放入到合并空间，并移动指针到下一位置；
4. 重复步骤 3 直到某一指针达到序列尾；
5. 将另一序列剩下的所有元素直接复制到合并序列尾。

## 代码实现

```java
import java.util.Arrays;

public class MergeSort {
    public int[] sort(int[] sourceArray) {
        // 开辟一个新的数组空间，防止在递归中多次创建
        int[] temp = new int[sourceArray.length];
        sortRecursively(sourceArray, temp, 0, sourceArray.length - 1);
        return sourceArray;
    }

    private void sortRecursively(int[] array, int[] copy, int low, int high) {
        if (low < high) {
            int mid = (low + high) >> 1;
            sortRecursively(array, copy, low, mid);
            sortRecursively(array, copy, mid + 1, high);
            merge(array, copy, low, high);
        }
    }

    private void merge(int[] array, int[] copy, int low, int high) {
        int mid = (low + high) >> 1;
        // 第一个数组的起始位置
        int i = low;
        // 第二个数组的起始位置
        int j = mid + 1;

        // copy 数组的起始位置
        int k = 0;
        while (i <= mid && j <= high) {
            if (array[i] < array[j]) {
                copy[k++] = array[i++];
            } else {
                copy[k++] = array[j++];
            }
        }

        // 若第一个数组仍然有数据，全部移动到 copy 数组中
        while (i <= mid) {
            copy[k++] = array[i++];
        }
        // 若第二个数组仍然有数据，全部移动到 copy 数组中
        while (j <= high) {
            copy[k++] = array[j++];
        }

        // 将 copy 数组的数据复制到原数组
        k = 0;
        while (low <= high) {
            array[low++] = copy[k++];
        }
    }

    public static void main(String[] args) {
        MergeSort s = new MergeSort();
        int[] array = {3, 5, 3, 7, 14, 6, 11, 5};
        System.out.println(Arrays.toString(s.sort(array)));
    }
}
```

## 算法分析
时间复杂度：从上文的图中可看出，每次合并操作的平均时间复杂度为 O(n)，而完全二叉树的深度为 log2n。总的平均时间复杂度为 O(nlogn)。而且，归并排序的最好，最坏，平均时间复杂度均为 O(nlogn)。
空间复杂度：开辟了一个与原数据相同长度的临时数组，空间复杂度为 O(n)。
稳定性：合并过程中我们可以保证如果两个当前元素相等时，我们把处在前面的序列的元素保存在结 果序列的前面，这样就保证了稳定性，因此归并排序是稳定排序。
