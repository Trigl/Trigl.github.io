---
layout:     post
title:      "「剑指 Offer」面试题 4：替换空格"
date:       2018-11-27 01:00:00
author:     "Ink Bai"
header-img: "/img/post/offer-sword-4.jpg"
tags:
    - 面试
---

> 实现一个函数，把字符串中的每个空格替换成 `%20`。例如输入 `We are happy.`，则输出 `We%20are%20happy.`。

这个题目很简单，不用 `replace` 函数来做直接进行替换有两种方式。
第一种是从前往后依次遍历，每次遇到空格把空格替换成 `%20`，然后把空格后面的元素全部后移 2 位，这样的时间复杂度是 `O(n²)`。
第二种方式是从后往前遍历，首先计算出替换后字符串的长度，然后给该字符串设置这个新的长度值，之后依次遍历填充新的字符即可，这样的时间复杂度是 `O(n)`，我们采用这种方式来实现。

```java
public class Solution {
    public static String replaceSpace(StringBuffer str) {
        if(str == null || str.toString().equals("")) {
            return "";
        }

        // 计算空格的数量
        int oldLength = str.length();
        int spaceNum = 0;
        for(int i = 0; i < oldLength; i++) {
            if(str.charAt(i) == ' ') {
                spaceNum++;
            }
        }

        // 替换后的字符串的长度
        int newLength = oldLength + spaceNum * 2;
        str.setLength(newLength);
        newLength--;
        // 从后往前遍历旧字符串的每一个字符并且替换
        for(int i = oldLength - 1; i >= 0; i--) {
            if(str.charAt(i) == ' ') {
                str.setCharAt(newLength--, '0');
                str.setCharAt(newLength--, '2');
                str.setCharAt(newLength--, '%');
            } else {
                str.setCharAt(newLength--, str.charAt(i));
            }
        }

        return str.toString();
    }

    // 测试代码
    public static void main(String[] args) {
        // 1. 输入的字符串中包含空格：空格位于最前面，最后面，中间，有连续多个空格
        StringBuffer sb1 = new StringBuffer(" Hello world,   I love you! ");
        System.out.println(replaceSpace(sb1));

        // 2. 输入的字符串中没有空格
        StringBuffer sb2 = new StringBuffer("Hahahahaha!");
        System.out.println(replaceSpace(sb2));

        // 3. 特殊输入
        // 字符串是 null
        StringBuffer sb3 = null;
        System.out.println(replaceSpace(sb3));
        // 字符串是空字符串
        StringBuffer sb4 = new StringBuffer("");
        System.out.println(replaceSpace(sb4));
        // 字符串只有一个空格
        StringBuffer sb5 = new StringBuffer(" ");
        System.out.println(replaceSpace(sb5));
        // 字符串只有连续多个空格
        StringBuffer sb6 = new StringBuffer("    ");
        System.out.println(replaceSpace(sb6));
    }
}
```
