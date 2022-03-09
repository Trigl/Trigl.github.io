---
layout:     post
title:      "希尔排序"
date:       2019-01-29 04:00:00
author:     "Ink Bai"
catalog:    true
header-img: "/img/post/shell-sort.jpg"
tags:
    - 算法与数据结构
---

## 思想
希尔排序是希尔（Donald Shell）于1959年提出的一种排序算法。希尔排序也是一种插入排序，它是简单插入排序经过改进之后的一个更高效的版本，也称为缩小增量排序，同时该算法是冲破O(n2）的第一批算法之一。

希尔排序是基于插入排序的以下两点性质而提出改进方法的：

1. 插入排序在对几乎已经排好序的数据操作时，效率高，即可以达到线性排序的效率；
2. 但插入排序一般来说是低效的，因为插入排序每次只能将数据移动一位。

简单插入排序很循规蹈矩，不管数组分布是怎么样的，依然一步一步的对元素进行比较，移动，插入，比如 [5,4,3,2,1,0] 这种倒序序列，数组末端的0要回到首位置很是费劲，比较和移动元素均需 n-1 次。而希尔排序在数组中采用跳跃式分组的策略，通过某个增量将数组元素划分为若干组，然后分组进行插入排序，随后逐步缩小增量，继续按组进行插入排序操作，直至增量为1。希尔排序通过这种策略使得整个数组在初始阶段达到从宏观上看基本有序，小的基本在前，大的基本在后。然后缩小增量，到增量为1时，其实多数情况下只需微调即可，不会涉及过多的数据移动。

总结起来，希尔排序的基本思想是：先将整个待排序的记录序列分割成为若干子序列分别进行直接插入排序，待整个序列中的记录“基本有序”时，再对全体记录进行依次直接插入排序。

## 算法步骤

1. 选择增量 gap=length/2，缩小增量继续以 gap = gap/2 的方式，这种增量选择我们可以用一个序列来表示，{n/2,(n/2)/2...1}，称为增量序列。
2. 在每个 gap 下，分别进行插入排序，这样就会以 gap=n/2，gap=(n/2)/2，...，gap=1 的顺序分别插入排序，在 gap=1 的时候也就实现了全排序。

![](/img/content/shell-sort.jpg)

## 代码实现

```java
import java.util.Arrays;

public class ShellSort {
    public int[] sort(int[] sourceArray) {
        // 不改变原数组，在复制数组上操作
        int[] arr = Arrays.copyOf(sourceArray, sourceArray.length);

        // 增量 gap，并逐步缩小增量
        for (int gap = arr.length / 2; gap > 0; gap /= 2) {
            // 对每一个 gap 的分组进行插入排序
            for (int i = gap; i < arr.length; i++) {
                int j = i;
                while (j - gap > 0 && arr[j] < arr[j - gap]) {
                    swap(arr, j, j - gap);
                    j -= gap;
                }
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
        ShellSort s = new ShellSort();
        int[] array = {3, 5, 3, 7, 14, 6, 11, 5};
        System.out.println(Arrays.toString(s.sort(array)));
    }
}
```

## 算法分析
时间复杂度：虽然希尔排序是基于插入排序的优化，但是我们上面选择的增量序列{n/2,(n/2)/2...1}(希尔增量)，其最坏时间复杂度依然为 O(n^2)，一些经过优化的增量序列如 Hibbard 经过复杂证明可使得最坏时间复杂度为O(n^3/2)。
空间复杂度：O(1)。
稳定性：稳定的排序算法。
