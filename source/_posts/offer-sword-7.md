---
layout:     post
title:      "「剑指 Offer」面试题 7：用两个栈实现队列"
date:       2018-11-29 02:00:00
author:     "Ink Bai"
header-img: "/img/post/offer-sword-7.jpg"
catalog:    true
tags:
    剑指 Offer
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

算法题目特别要关注异常情况，这里如果队列为空的时候我们调用 pop 方法怎么处理呢？
队列为空说明 stack1 和 stack2 都没有元素，这个时候是不会返回任何值的，需要抛出一个异常。这个时候如果抛出 `Exception` 的会报错的，必须抛出 `RuntimeException`，这是为什么呢，我们需要了解 Exception 和 RuntimeException 的区别：

- Exception：在程序中必须使用 try...catch 进行处理。
- RuntimeException：可以不使用 try...catch 进行处理，但是如果有异常产生，则异常将由 JVM 进行处理。

对于 RuntimeException 的子类最好也使用异常处理机制。虽然 RuntimeException 的异常可以不使用 try...catch 进行处理，但是如果一旦发生异常，则肯定会导致程序中断执行，所以，为了保证程序再出错后依然可以执行，在开发代码时最好使用 try...catch 的异常处理机制进行处理。
