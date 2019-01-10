---
layout:     post
title:      "「剑指 Offer」面试题 21：包含 min 函数的栈"
date:       2019-01-10 05:00:00
author:     "Ink Bai"
header-img: "/img/post/offer-sword-21.jpg"
tags:
    - 剑指 Offer
---

> 定义栈的数据结构，请在该类型中实现一个能够得到栈的最小元素的 min 函数。在该栈中，调用 min、push 及 pop 的时间复杂度都是 O(1)。

引入一个辅助栈用来存放当前位置的最小元素，入栈时如果该数小于这个最小值，那么辅助栈就入这个数，如果大于该最小值，就还是入之前的元素，出栈的时候同时出即可。

代码如下：

```java
import java.util.Stack;

public class Solution {
    Stack<Integer> stack = new Stack<Integer>();
    Stack<Integer> minStack = new Stack<Integer>();
    public void push(int node) {
        stack.push(node);
        if (minStack.size() == 0 || node < minStack.peek()) {
            minStack.push(node);
        } else {
            minStack.push(minStack.peek());
        }
    }

    public void pop() {
        stack.pop();
        minStack.pop();
    }

    public int top() {
        return stack.peek();
    }

    public int min() {
        return minStack.peek();
    }
}
```
