---
layout:     post
title:      "「剑指 Offer」面试题 22：栈的压入、弹出序列"
date:       2019-01-10 06:00:00
author:     "Ink Bai"
header-img: "/img/post/offer-sword-22.jpg"
tags:
    剑指 Offer
---

> 输入两个整数序列，第一个序列表示栈的压入顺序，请判断第二个序列是否为该栈的弹出顺序。假设压入栈的所有数字均不相等。例如序列 1、2、3、4、5 是某栈的压栈序列，序列 4、5、3、2、1 是该压栈序列对应的一个弹出序列，但 4、3、5、1、2 就不可能是该压栈序列的弹出序列。

定义一个辅助栈用来模拟入栈出栈的过程。首先一次将压栈序列中的树压入，在此过程中比较弹出栈中的元素和栈顶元素是否相等，相等的话就弹出，不相等的话继续压入，直到将所有元素压入完毕，然后看栈中的元素是否全部弹出，是的话说明是弹出序列，否则不是。

代码如下：

```java
import java.util.Stack;

public class Solution {
    public boolean IsPopOrder(int [] pushA,int [] popA) {
      if(pushA == null || popA == null || pushA.length == 0 || popA.length == 0 || pushA.length != popA.length) {
          return false;
      }

        int popIndex = 0;
        Stack<Integer> stack = new Stack<Integer>();
        for (int i = 0; i < pushA.length; i++) {
            stack.push(pushA[i]);
            while (popIndex < popA.length && popA[popIndex] == stack.peek()) {
                stack.pop();
                popIndex++;
            }
        }
        return stack.empty();
    }
}
```
