---
layout:     post
title:      "「剑指 Offer」面试题 7：用两个栈实现队列"
date:       2018-11-29 02:00:00
author:     "Ink Bai"
header-img: "/img/post/offer-sword-7.jpg"
tags:
    - 面试
---

> 用两个栈来实现一个队列，完成队列的Push和Pop操作。 队列中的元素为int类型。

队列特点是先进先出，栈的特点是先进后出，所以我们需要两个栈来达到负负为正的效果，即元素先进入第一个栈，再进入第二个栈，然后再出来，最终的顺序仍然是先进先出。

```java
import java.util.Stack;

public class Solution {
    Stack<Integer> stack1 = new Stack<Integer>();
    Stack<Integer> stack2 = new Stack<Integer>();

    public void push(int node) {
        stack1.push(node);
    }

    public int pop() {
        if (stack1.isEmpty() && stack2.isEmpty()) {
            throw new RuntimeException("Queue is empty!");
        }
        // 当 stack2 为空时才会取出 stack1 的元素
        if (stack2.isEmpty()) {
            while (!stack1.isEmpty()) {
                stack2.push(stack1.pop());
            }
        }
        return stack2.pop();
    }
}
```
