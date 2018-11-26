---
layout:     post
title:      "「剑指 Offer」面试题 4：替换空格"
date:       2018-11-27 01:00:00
author:     "Ink Bai"
header-img: "/img/post/offer-sword-4.jpg"
tags:
    - 剑指 Offer
---

> 实现一个函数，把字符串中的每个空格替换成 `%20`。例如输入 `We are happy.`，则输出 `We%20are%20happy.`。

这个题目很简单，不用 `replace` 函数来做直接进行替换有两种方式。
第一种是从前往后依次遍历，每次遇到空格把空格替换成 `%20`，然后把空格后面的元素全部后移 2 位，这样的时间复杂度是 `O(n²)`。
第二种方式是从后往前遍历，首先计算出替换后字符串的长度，然后给该字符串设置这个新的长度值，之后依次遍历填充新的字符即可，这样的时间复杂度是 `O(n)`，我们采用这种方式来实现。

```java
public class Solution {
    public String replaceSpace(StringBuffer str) {
        // 空格数
    	int spaceNum = 0;
        // 计算空格数
        for(int i = 0; i < str.length(); i++) {
            if(str.charAt(i) == ' ') {
                spaceNum++;
            }
        }

        // 设置新字符串长度
        int newLength = str.length() + spaceNum * 2;
        // 设置从后遍历对应的新老字符串的下标
        int newIndex = newLength - 1;
        int oldIndex = str.length() - 1;
        str.setLength(newLength);
        for(; oldIndex >= 0; oldIndex--) {
            if(str.charAt(oldIndex) == ' ') {
                str.setCharAt(newIndex--, '0');
                str.setCharAt(newIndex--, '2');
                str.setCharAt(newIndex--, '%');
            } else {
                str.setCharAt(newIndex--, str.charAt(oldIndex));
            }
        }
        return str.toString();
    }
}
```
