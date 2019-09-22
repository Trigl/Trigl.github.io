---
layout:     post
title:      "闲来无事刷刷题"
date:       2019-02-11 01:00:00
author:     "Ink Bai"
catalog:    true
header-img: "/img/post/interview-algorithm.jpg"
tags:
    面试
---
> 刷题呢可以活跃思维，巩固基础，当然最重要的是面试要考啦咳咳。
首先需要熟练掌握链表、树、栈、队列和哈希表等数据结构，以及它们的操作。
一般最经常考察链表和二叉树，链表的插入和删除结点，二叉树的各种遍历方法的循环和递归写法。
掌握常用查找排序算法，重点是二分查找、归并排序和快速排序。
更加难一点的要求熟练掌握动态规划和贪婪算法。

## 查找与排序
**十大排序算法**
[冒泡排序](http://baixin.ink/2019/01/29/bubble-sort/)
[选择排序](http://baixin.ink/2019/01/29/sellection-sort/)
[插入排序](http://baixin.ink/2019/01/29/insertion-sort/)
[希尔排序](http://baixin.ink/2019/01/29/shell-sort/)
[归并排序](http://baixin.ink/2019/01/29/merge-sort/)
[快速排序](http://baixin.ink/2019/01/29/quick-sort/)

**四大查找算法**
顺序查找
二分查找
哈希表查找
二叉树查找

## 数组
数组中的内存是连续的，可以根据下标在 O(1) 时间内读写任何元素，因此它的时间效率很高。我们可以根据数组时间效率高的优点，用数组来实现简单的哈希表：把数组的下标设为哈希表的键值（key），而把数组中的每一个元素设为哈希表的值（value），这样我们就可以在 O(1) 的时间内实现查找。

**剑指 offer**
[面试题 03：二维数组中的查找](http://baixin.ink/2018/11/23/offer-sword-3/)
[面试题 08：旋转数组的最小数字](http://baixin.ink/2019/01/01/offer-sword-8/)
[面试题 14：调整数组顺序使奇数位于偶数前面](http://baixin.ink/2019/01/08/offer-sword-14/)

## 字符串
**剑指 offer**
[面试题 04：替换空格](http://baixin.ink/2018/11/27/offer-sword-4/)

## 链表
链表问题一般会有大量的指针移动操作，当我们使用一个指针解决不了的时候可以多定义几个指针来解决。

**剑指 offer**
[面试题 05：从尾到头打印链表](http://baixin.ink/2018/11/27/offer-sword-5/)
[面试题 13：在 O(1) 时间删除链表结点](http://baixin.ink/2019/01/08/offer-sword-13/)
[面试题 15：链表中倒数第 k 个结点](http://baixin.ink/2019/01/08/offer-sword-15/)
[面试题 16：反转链表](http://baixin.ink/2019/01/09/offer-sword-16/)
[面试题 17：合并两个排序的链表](http://baixin.ink/2019/01/09/offer-sword-17/)
[面试题 26：复杂链表的复制](http://baixin.ink/2019/03/21/offer-sword-26/)

## 树
树有三种遍历方式：

- 前序遍历：根结点 -> 左子结点 -> 右子结点
- 中序遍历：左子结点 -> 根节点 -> 右子结点
- 后续遍历：左子结点 -> 右子结点 -> 根结点

宽度优先遍历：从上到下，从左到右遍历树。

二叉搜索树：左子结点总是小于或等于根节点，右子结点总是大于或等于根节点。我们可以平均在 O(logn) 的时间内根据数值在二叉搜索树中找到一个结点。

堆：最大堆和最小堆。最大堆中根节点的值最大，在最小堆中根节点的值最小。很多需要快速找到最大值或者最小值的问题都可以用堆来解决。

红黑树：树中的结点定义为红、黑两种颜色，并通过规则确保从根节点到叶结点的最长路径的长度不超过最短路径的两倍。

**剑指 offer**
[面试题 06：重建二叉树](http://baixin.ink/2018/11/29/offer-sword-6/)
[面试题 18：树的子结构](http://baixin.ink/2019/01/10/offer-sword-18/)
[面试题 19：二叉树的镜像](http://baixin.ink/2019/01/10/offer-sword-19/)
[面试题 24：二叉搜索树的后序遍历序列](http://baixin.ink/2019/01/11/offer-sword-24/)
[面试题 25：二叉树中和为某一值的路径](http://baixin.ink/2019/03/11/offer-sword-25/)
[面试题 27：二叉搜索树与双向链表](http://baixin.ink/2019/04/06/offer-sword-27/)

## 栈和队列
**剑指 offer**
[面试题 07：用两个栈实现队列](http://baixin.ink/2018/11/29/offer-sword-7/)
[面试题 21：包含 min 函数的栈](http://baixin.ink/2019/01/10/offer-sword-21/)
[面试题 22：栈的压入、弹出序列](http://baixin.ink/2019/01/10/offer-sword-22/)
[面试题 23：从上往下打印二叉树](http://baixin.ink/2019/01/11/offer-sword-23/)

## 位运算
**剑指 offer**
[面试题 10：二进制中 1 的个数](http://baixin.ink/2019/01/07/offer-sword-10/)

## 回溯法
**剑指 offer**

## 动态规划
**剑指 offer**

## 大数据算法
**剑指 offer**

## 其他
**剑指 offer**
[面试题 09：斐波那契数列](http://baixin.ink/2019/01/02/offer-sword-9/)
[面试题 11：数值的整数次方](http://baixin.ink/2019/01/07/offer-sword-11/)
[面试题 12：打印 1 到最大的 n 位数](http://baixin.ink/2019/01/08/offer-sword-12/)
[面试题 20：顺时针打印矩阵](http://baixin.ink/2019/01/10/offer-sword-20/)

## Refer
[十大经典排序算法](https://github.com/hustcc/JS-Sorting-Algorithm)
[面试中的排序算法总结](https://www.cnblogs.com/wxisme/p/5243631.html)
[算法总结](https://www.cnblogs.com/chengxiao/category/880910.html)
[关于快速排序的四种写法](https://segmentfault.com/a/1190000004410119)
[查找算法的Java实现](https://www.jianshu.com/p/b07c69a91535)
[七大查找算法](http://www.cnblogs.com/maybe2030/p/4715035.html)
[八大排序算法稳定性分析](https://zhuanlan.zhihu.com/p/36120420)
