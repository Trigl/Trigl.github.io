---
layout:     post
title:      "「Notes」Java 并发编程实战"
date:       2020-11-30
author:     "Ink Bai"
header-style: "text"
catalog:    true
tags:
    - 并发
---

> 2020.11.29 开始读，每天读一章，先读一周试试水。

## 非原子的64位操作
Java 内存模型要求，变量的读取和写入操作都必须是原子操作，但对于非 volatile 类型的 long 和 double 变量，JVM 允许将 64 位的读操作或写操作分解为两个 32 位的操作。

## volatile 作用
- 编译器和运行时不再重排序
- 不会被缓存到寄存器或者对其他处理器不可见的地方

## 不要在构造函数中使 this 引用逸出
发布一个对象地内部类实例也会间接地发布这个对象：

```Java
public class ThisEscape {
    public ThisEscape(EventSource source) {
        source.registerListener(
                new EventListener() {
                    public void onEvent(Event e) {
                        doSomething(e);
                    }
                }
        );
    }
}
```

这里当 source 得到内部类 EventListener 的实例时，source 可能会间接地修改 ThisEscape 对象，因此 ThisEscape 也逸出了。

在对象的构造函数中发布一个对象，只会发布一个尚未构造完成的对象，因为其他外部对象可能会修改这个对象，这种对象就被认为是不正确构造。

类似地，在构造函数中创建线程，this 引用都会被新创建的线程共享。此时最好不要启动它，而是在构造函数完成之后，通过一个 start 或者 initialize 方法来启动。

## final + volatile 实现线程安全
每当需要对一组相关数据以原子方式执行某个操作时，就可以考虑创建一个不可变的类来包含这些数据，保证不可变性，同时使用一个 volatile 类型的引用来确保可见性，结合起来可以实现线程安全。

如下面因式分解的例子：

```java
@Immutable
class OneValueCache {
    private final BigInteger lastNumber;
    private final BigInteger[] lastFactors;

    public OneValueCache(BigInteger i, BigInteger[] factors) {
        lastNumber = i;
        lastFactors = factors;
    }

    public BigInteger[] getFactors(BigInteger i) {
        if (lastNumber == null || !lastNumber.equals(i)) {
            return null;
        } else {
            return Arrays.copyOf(lastFactors, lastFactors.length);
        }
    }
}


@ThreadSafe
public class VolatileCachedFactorizer implements Servlet {
    private volatile OneValueCache cache = new OneValueCache(null, null);

    public void service(ServletRequest req, ServletResponse resp) {
        BigInteger i = extractFromRequest(req);
        BigInteger[] factors = cache.getFactors(i);
        if (factors == null) {
            factors = factor(i);
            cache = new OneValueCache(i, factors);
        }
        encodeIntoResponse(resp, factors);
    }
}
```

不可变性对象需满足：

- 状态不可修改
- 成员变量都是 final 类型
- 正确的创建过程（创建期间，this 引用不能逸出）

## 不可变对象的初始化安全性
一般对象初始化以后，当另一个线程访问这个对象时，如果没有加同步，那么该对象的状态对这个线程不一定可见。

但是对于不可变对象，Java 内存模型为它的共享提供了一种特殊的初始化安全性保证，任何线程都可以在不需要额外同步的情况下安全地访问不可变对象，即使在发布这些对象时没有使用同步，这种保证还将延伸到被正确创建对象中所有 final 类型的成员变量中。

当然，如果 final 类型的成员变量指向的是一个可变对象，那么访问这个对象的状态仍然需要同步。
