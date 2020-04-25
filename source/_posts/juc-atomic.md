---
layout:     post
title:      "AtomicInteger 源码阅读"
date:       2020-04-25
author:     "Ink Bai"
catalog:    true
header-img: "/img/post/juc-atomic.jpg"
tags:
    - 并发
    - JDK 源码
---

## 预备知识：CAS
AtomicInteger 的核心都是基于 CAS 的，那么 CAS 是什么呢？让我们先从悲观锁和乐观锁说起。

#### 悲观锁和乐观锁
一般来说乐观的人对生活总是充满希望，遇到任何事情都会说 everything will be ok! 而悲观的人呢，总是会假定事情会想着不好的方向发展，做最坏的打算。

对应到程序中的并发情况，也就有两种锁，悲观锁和乐观锁。

悲观锁总是会假设最坏的情况，每次去操作一份数据时，必然有其他人也会同时操作，所以每次操作数据都要上锁。传统数据库中的行锁、写锁，Java 中的 `synchronized` 和 `ReentrantLock` 等独占锁就是使用了悲观锁。

乐观锁总是假设最好的情况，每次去操作数据时别人都不会去修改，所以不会上锁，但是乐观并不代表憨憨，为了保险起见在操作数据的时候还是会先判断一下在此期间有没有其他人更新这个数据，如果有人已经修改了，那我就在你修改的基础上重新再去更新，这就是 CAS 的来源，也就是 `compareAndSwap`。

#### 两种锁的优缺点和适用场景
从上面分析可以知道，乐观锁总是假设自己修改的时候别人没有修改，如果别人也在修改的话就需要重新比较和更新，所以乐观锁适用于同时并发修改少的情况，也就是读多写少的情况，在这种情况下，写冲突很少，省去了锁的开销，整个系统的性能很高。但是如果写冲突很多的话，在写的时候会经常发生冲突，需要不停进行 CAS 操作，这样反而降低了性能，不如直接加锁来的快速与高效，因此在多写的场景下适用悲观锁反而更合适。

#### 乐观锁常见的实现方式
**1、版本号机制**
在操作或者查询数据时会对应加上一个版本号字段，当更新数据的同时还要将版本号+1，这样当其他人修改这个字段时，发现自己的版本号是落后于正式版本号的，就会先查询最新的数据和版本号，然后再更新，避免了并发问题。

**2、CAS**
CAS 是 CPU 提供的一种原子操作，可以在无锁情况下实现多线程之间的变量同步，主要有三个操作数：

- 需要读写的内存之 V
- 进行比较的旧值 A
- 要写入的新值 B

当且仅当 V 的值等于 A 时，CAS 通过原子方式用新值 B 来更新 V 的值。如果 V 的值不等于 A，就先重新获取 A，然后再用 CAS 更新新值 B，这种操作称为自旋。

#### CAS 的缺点
**1、ABA 问题**
如果一个变量 V 开始是 A，后面改成了 B，然后又改成了 A。在此期间我们检查它的值的时候，发现一直都是 A，就会认为这个变量没有发生改变，但实际上是发生过改变的，这就是 CAS 中的 ABA 问题。

解决 ABA 问题也很简单，那就是添加版本号，例如 `AtomicStampedReference` 这个类就是通过把 stamp 作为版本号来防止 ABA 问题，这个类的具体情况后面再讲。

**2、多写情况下循环时间长**
自旋 CAS 如果长时间不成功会一直循环，这会给 CPU 带来非常大的执行开销。一般解决方式有两种：

- 多写情况不再使用 CAS，而是使用悲观锁的思想，即直接上锁。
- 将对一个变量的多写分摊到多个变量上，减少冲突，JDK `LongAdder` 这个类就是使用了这种思想，现在先按下不表，后面安排。


## AtomicInteger 详解
AtomicInteger 类是为了保证多线程情况下保证对 Integer 的操作都是原子的，不会产生并发问题。先来看一下如何使用 AtomicInteger：

```java
AtomicInteger i = new AtomicInteger(0);
i.addAndGet(3); // 3
i.getAndAdd(3); // 3
i.get(); // 6
```

接着开始看源码，首先看开头部分：

```java
// setup to use Unsafe.compareAndSwapInt for updates
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long valueOffset;

static {
    try {
        valueOffset = unsafe.objectFieldOffset
            (AtomicInteger.class.getDeclaredField("value"));
    } catch (Exception ex) { throw new Error(ex); }
}

private volatile int value;
```

这部分内容的关键是 Unsafe 类的使用，这个类完全可以另开一篇单独讲了，就单独在 AtomicInteger 里，Unsafe 最主要的作用就是提供可以执行 CAS 的本地方法。

- **Line 5** - 这部分代码放在可 static 静态代码块，确保在初始化对象前就率先执行。
- **Line 7** - 通过反射机制来计算字段 value 相对于 AtomicInteger 对象的偏移量，这里计算偏移量的目的是为了后面可以计算出 value 字段对应的内存地址。
- **Line 12** - value 通过 volatile 来保证可见性，另外通过 CAS 操作确保原子性，如此来保证 AtomicInteger 的并发安全性。

Unsafe 提供的 CAS 本地方法是这样的，AtomicInteger 内提供的几乎所有方法都是基于这个本地方法：

```java
/* 该方法来自 Unsafe 类 */
public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
```

这里的方法参数命名很不友好，我们看一下 AtomicInteger 里面是怎么使用这个方法的：

```java
public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```

这个方法就是提供一个预期值 except 和一个待更新值 update，通过 CAS 比较预期值，如果预期值与内存中 value 的值一致的话，就将 value 的值更新为 update。value 的内存地址就是通过对象的内存地址（this 获得）和上面讲到的 value 字段相对于对象的偏移量得到的。

上面 `compareAndSet` 方法仅会执行一次，返回一个 boolean 类型变量标志成功还是失败，如果返回 false 那就意味着值更新失败了，所以我们一般会使用 `getAndIncrement` 来递增，它内部是怎么实现的呢：

```java
/* 位于 AtomicInteger 内，获取值然后再+1 */
public final int getAndIncrement() {
    return unsafe.getAndAddInt(this, valueOffset, 1);
}

/* 位于 Unsafe 内，经典的 CAS 自旋操作 */
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
}
```

AtomicInteger JDK1.8 之后新添加了包含函数式编程的方法：

```java
public final int getAndUpdate(IntUnaryOperator updateFunction) {
    int prev, next;
    do {
        prev = get();
        next = updateFunction.applyAsInt(prev);
    } while (!compareAndSet(prev, next));
    return prev;
}

public final int getAndAccumulate(int x,
                                  IntBinaryOperator accumulatorFunction) {
    int prev, next;
    do {
        prev = get();
        next = accumulatorFunction.applyAsInt(prev, x);
    } while (!compareAndSet(prev, next));
    return prev;
}
```

这两个方法的作用是对一元数和二元数做转换：

- **Line 1** - IntUnaryOperator 是一元函数式接口，它提供了一个 `applyAsInt(int operand)` 可以对单一 int 类型数据进行转换。
- **Line 2** - 相似的，IntBinaryOperator 是一个二元函数式接口，提供了 `applyAsInt(int left, int right)` 对二元 int 类型数据进行转换。

上面两个方法的使用示例如下：

```java
AtomicInteger i = new AtomicInteger(9);
i.updateAndGet(n -> n % 2 == 0 ? 0 : 1); // 1
i.accumulateAndGet(3, (a, b) -> a + b); // 4
```

## AtomicStampedReference 类解决 ABA 问题
AtomicStampedReference 类内置了一个数据结构 Pair，这个 Pair 里面添加了字段 stamp 作为版本号：

```java
private static class Pair<T> {
    final T reference;
    final int stamp;
    private Pair(T reference, int stamp) {
        this.reference = reference;
        this.stamp = stamp;
    }
    static <T> Pair<T> of(T reference, int stamp) {
        return new Pair<T>(reference, stamp);
    }
}

private volatile Pair<V> pair;
```

因此每次更新 pair 的时候，除了更新 reference，还需要更新版本号字段 stamp。

## 总结
本文首先介绍了悲观锁和乐观锁的区别，然后引出了大名鼎鼎的 CAS 无锁算法，然后详细讲解 AtomicInteger 是如何基于 CAS 算法实现原子性的，最后又介绍了 AtomicStampedReference 如何解决 CAS 中的 ABA 问题，相信通过上面的讲解，我们已经可以在实际编程中灵活应用原子类。

## Refer
[面试必备之乐观锁与悲观锁](https://juejin.im/post/5b4977ae5188251b146b2fc8)
[java-juc-原子类-AtomicInteger初探](https://cruise1008.github.io/2016/04/01/java-juc-%E5%8E%9F%E5%AD%90%E7%B1%BB-AtomicInteger%E5%88%9D%E6%8E%A2/)
