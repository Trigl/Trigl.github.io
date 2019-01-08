---
layout:     post
title:      "「剑指 Offer」面试题 12：打印 1 到最大的 n 位数"
date:       2019-01-08 02:00:00
author:     "Ink Bai"
header-img: "/img/post/offer-sword-12.jpg"
tags:
    - 剑指 Offer
---

> 输入数字 n，按顺序打印出从 1 到最大的 n 位十进制数。比如输入 3，则打印出 1、2、3 一直到最大的 3 位数即 999。

这道题目乍一看好像很简单，就是打印数字嘛，其实里面有一个很重要的坑，那就是大数问题，即这里面的数字可能极其大，超过计算机存储整数的最大范围。

因此我们需要找到一种合适的方式来表达一个大数，最常用也最容易的方法就是用字符串或者数组表示大数，我们这里用数组的方式来解决这个问题。定义一个长度为 n 的数组，数字的个位就对应到数组的最后一位，如果实际数字不够 n 位，数组的前半部分补 0。那么我们思考一下如何用一个数组表示所有十进制数的可能情况呢？这不正是这个数组中 n 个元素分别从 0 到 9 的全排列吗？也就是说，我们把数字的每一位都从 0 到 9 排列一遍，就得到了所有的十进制数。只是我们在打印的时候，数字排在前面的 0 我们不打印出来罢了。

全排列用递归很容易表达，数字的每一位都可能是 0～9 中的一个数，然后设置下一位。递归结束的条件是我们已经设置了数字的最后一位。

具体代码如下：

```java
public class Solution {
    public static void printOneToNthDigits(int n) {
        if (n < 1) {
            throw new RuntimeException("The input number must be larger than 0!");
        }
        //创建一个数组用于存放打印结果
        int[] arr = new int[n];
        printOneToNthDigitsRecursively(0, arr);
    }

    /**
     * @param n   当前处理的是第n个元素，从0开始计数
     * @param arr 存放结果的数组
     */
    private static void printOneToNthDigitsRecursively(int n, int[] arr) {
        // 说明所有的数据排列选择都已经处理完了
        if (n >= arr.length) {
            printArray(arr);
        } else {
            for (int i = 0; i <= 9; i++) {
                arr[n] = i;
                printOneToNthDigitsRecursively(n + 1, arr);
            }
        }
    }

    private static void printArray(int[] arr) {
        // 找第一个非0的元素
        int index = 0;
        while (index < arr.length && arr[index] == 0) {
            index++;
        }
        // 从第1个非0元素开始，输出到最后一个元素
        for (int i = index; i < arr.length; i++) {
            System.out.print(arr[i]);
        }
        // 条件成立，说明数组中有非0元素，所以需要换行
        if (index < arr.length) {
            System.out.println();
        }
    }

    public static void main(String[] args) {
        printOneToNthDigits(3);
    }
}
```
