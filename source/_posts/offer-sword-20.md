---
layout:     post
title:      "「剑指 Offer」面试题 20：顺时针打印矩阵"
date:       2019-01-10 04:00:00
author:     "Ink Bai"
header-img: "/img/post/offer-sword-20.jpg"
tags:
    - 剑指 Offer
---

> 输入一个矩阵，按照从外向里以顺时针的顺序依次打印出每一个数字。

例如：如果输入如下矩阵

```
1    2    3    4
5    6    7    8
9    10   11   12
13   14   15   16
```

则依次打印出 1、2、3、4、8、12、16、15、14、13、9、5、6、7、11、10。

代码如下：

```java
import java.util.ArrayList;
public class Solution {
    public ArrayList<Integer> printMatrix(int [][] matrix) {
        ArrayList<Integer> list = new ArrayList<Integer>();
       if (matrix != null && matrix.length > 0) {
           // matrix not null and row > 0
           int row = matrix.length;
           int column = matrix[0].length;
           int start = 0;
           if (column > 0) {
               while (row > start * 2 && column > start * 2) {
                   list.addAll(printMatrixInCircle(matrix, row, column, start));
                   start++;
               }
           }
       }
        return list;
    }

    private ArrayList<Integer> printMatrixInCircle(int [][] matrix, int row, int column, int start) {
        ArrayList<Integer> list = new ArrayList<Integer>();
        int endX = column - start - 1;
        int endY = row - start - 1;

        // left to right
        for (int i = start; i <= endX; i++) {
            list.add(matrix[start][i]);
        }

        // up to down
        if (endY > start) {
            for (int i = start + 1; i <= endY; i++) {
                list.add(matrix[i][endX]);
            }
        }

        // right to left
        if (endY > start && endX > start) {
            for (int i = endX - 1; i >= start; i--) {
                list.add(matrix[endY][i]);
            }
        }

        // down to up
        if (endX > start && endY > start + 1) {
            for (int i = endY - 1; i >= start + 1; i--) {
                list.add(matrix[i][start]);
            }
        }
        return list;
    }
}
```
