---
layout:     post
title:      "LongAdder 源码阅读"
date:       2020-05-13
author:     "Ink Bai"
catalog:    true
header-style: "text"
tags:
    - 并发
    - JDK 源码
---

> [上一篇](https://baixin.ink/2020/04/25/juc-atomic/)讲解了原子类 AtomicInteger 的源码，它可以看作是一个线程安全的计数器，但是如果在高并发情况下 AtomicInteger 的性能就捉襟见肘了，因为在高并发情况下多个线程同时进行 CAS 操作会频繁地失败重试，CPU 损耗很高，这个时候我们就可以使用性能更加高的 LongAdder 来实现计数功能，本文将会深入解析为什么 LongAdder 的性能比 AtomicInteger 高。

## LongAdder 的使用
LongAdder 一般用作高并发情况下计数器，用法很简单：

```java
LongAdder la = new LongAdder();
// thread 1
la.add(1.5);
// thread 2
la.add(2.6);
// thread 3
la.add(3.3);
// get sum
la.sum();
```

## Striped64 的设计原理
LongAdder 继承了 Striped64，真正高性能的事情都是 Striped64 实现的。Striped64 是一个扩展自 Number 的包级私有类，主要用于在 64 位值上支持动态 striping 的类的通用表示和机制。

首先，我们明确什么是 striping（条带化）？

大多数磁盘系统都对访问次数（每秒的 I/O 操作，IOPS）和数据传输率（每秒传输的数据量，TPS）有限制。当达到这些限制时，后续需要访问磁盘的进程就需要等待，这就是所谓的磁盘冲突。当多个进程同时访问一个磁盘时，可能会出现磁盘冲突。因此，避免磁盘冲突是优化 I/O 性能的一个重要目标。

条带（strip）是把连续的数据分割成相同大小的数据块，把每段数据分别写入到阵列中的不同磁盘上的方法。使用条带化技术使得多个进程同时访问数据的多个不同部分而不会造成磁盘冲突，而且在需要对这种数据进行顺序访问的时候可以获得最大程度上的 I/O 并行能力，从而获得非常好的性能。

Striped64 正是利用条带化的设计理念，将逻辑上连续的数据分割为 64bit 的片段，并结合缓存行填充，减少高并发下 CAS 操作的竞争，从而提高并发更新的吞吐量。

在低并发情况下，线程间竞争比较低，这个时候 AtomicInteger 的 CAS 操作不会有很多的重试，性能上是 OK 的。但是如果在高并发场景下，多个线程同时进行 CAS，就会导致各自的自旋 CAS 重试次数增多，单个线程进行多次 CAS 也就意味着会占用更多的 CPU 时间片，整体性能就会下降很多。

所以高并发的瓶颈就在多个线程的大量竞争，Striped64 如何解决的这个问题呢，就是分散热点，减少竞争，将原本多个线程对同一个变量的竞争分摊到一个数组中，各个线程只对自己槽中的那个值进行 CAS 操作，这样热点就被分散了，大大减少了各个线程自旋 CAS 的次数，冲突的概率就小很多，一种典型的通过空间换取时间的方法。如果要获取真正的 long 值，只要将各个槽中的变量值累加返回。

![](/img/content/AfQfeeF.png)

## Striped64 核心数据结构
分散热点的核心数据结构如下：

```java
transient volatile Cell[] cells;

transient volatile long base;

transient volatile int cellsBusy;
```

被 transient 修饰的变量不会被序列化，这就意味这 cells 变量只会在内存中被使用，数组 cells 是实现高性能的关键。

AtomicInteger 只有一个 value，所有线程累加都要通过 CAS 竞争 value 这一个变量，高并发下线程争用非常严重。

而 Striped64 则有多个值用于累加，其中一个就是 base，它的作用类似于 AtomicInteger 里面的 value，在没有竞争的时候不会用到 cells 数组，cells 数组为空，只会对 base 进行累加，所以说并发不高的情况下 Striped64 的性能和 AtomicInteger 是持平的。

当高并发情况下有了竞争后，cells 数组就要上场表演了，它的作用可以看作是一个哈希表，当有多个线程竞争时，首先得到线程的哈希值，然后给该线程分配一个 cells 数组中的元素，那么该线程就只会竞争该数组中的一个元素了。

注意 cells 会从空数组开始逐渐扩容，每次扩容后的长度大小都是 2 的幂，并且最大长度不超过 CPU 的核数：

```java
/** Number of CPUS, to place bound on table size */
static final int NCPU = Runtime.getRuntime().availableProcessors();
```

扩容后的长度必须是 2 的幂，这一点我们后面再讲，而不超过 CPU 的核数其实很好理解，cells 数组要解决的是多个线程的竞争问题，而同时存在竞争的线程数最多也就是 CPU 的核数个，而这些线程会被哈希到 cells 数组中不同的位置，因此 cells 数组的最大长度等于 CPU 核数完全足够了，再大的话呢也可以，但没必要。

再来看一下成员变量 cellsBusy，它是一个自旋锁，它有两个值 0 或 1，它的作用是当要修改 cells 数组时加锁，防止多线程同时修改 cells 数组，0 为无锁，1 为加锁，加锁的状况有三种：

- cells 数组初始化的时候
- cells 数组扩容的时候
- 如果 cells 数组中某个元素为 null，给这个位置创建新的 Cell 对象的时候

这个地方没有必要使用阻塞锁，如果锁不可达，线程可以尝试其他的 slots，或者尝试 base 字段。在这些重试期间，竞争加剧，但是降低了局部性，这仍然比阻塞锁来得好。

## 什么是伪共享以及如何避免伪共享
首先看一下 Striped64 内部类 Cell 的实现：

```java
@sun.misc.Contended static final class Cell {
    volatile long value;
    Cell(long x) { value = x; }
    final boolean cas(long cmp, long val) {
        return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
    }

    // Unsafe mechanics
    private static final sun.misc.Unsafe UNSAFE;
    private static final long valueOffset;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> ak = Cell.class;
            valueOffset = UNSAFE.objectFieldOffset
                (ak.getDeclaredField("value"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
}
```

这个类很简单，final 类型，内部有一个 value 值，使用 CAS 来更新它的值；Cell 类唯一需要注意的地方就是注解 @sun.misc.Contended。

关于伪共享可以查看这篇文章：[一篇对伪共享、缓存行填充和CPU缓存讲的很透彻的文章](https://blog.csdn.net/qq_27680317/article/details/78486220)

## 累加过程
Striped64 提供了两种累加 API：longAccumulate 和 doubleAccumulate，两者的实现思路是一致的，只不过前者用于 long 值的累加，后者用于 double 值的累加。整个累加过程涉及初始化，调整大小，创建新 Cell，和/或争用的更新。下面我们以 longAccumulate 为例说明累加机制的实现原理，先看流程图：

![](/img/content/striped64.png)

源码如下，看长度就知道实现很复杂，有多个逻辑分支，这部分内容对照流程图大体看一下即可，重点是理解思路：

- if 表已初始化:
> if 映射到的槽是空的，加锁后再次判断，如果仍然是空的，初始化cell并关联到槽。
else if （槽不为空）在槽上之前的CAS已经失败，重试。
else if （槽不为空、且之前的CAS没失败，）在此槽的cell上尝试更新
else if 表已达到容量上限或被扩容了，重试。
else if 如果不存在冲突，则设置为存在冲突，重试。
else if 如果成功获取到锁，则扩容。
else 重散列，尝试其他槽。
- else if 锁空闲且获取锁成功，初始化表
- else if 回退 base 上更新且成功则退出
- else 继续

```java
final void longAccumulate(long x, LongBinaryOperator fn,
                          boolean wasUncontended) {
    int h;
    if ((h = getProbe()) == 0) {
        ThreadLocalRandom.current(); // force initialization
        h = getProbe();
        wasUncontended = true;
    }
    boolean collide = false;                // True if last slot nonempty
    for (;;) {
        Cell[] as; Cell a; int n; long v;
        if ((as = cells) != null && (n = as.length) > 0) {
            if ((a = as[(n - 1) & h]) == null) {
                if (cellsBusy == 0) {       // Try to attach new Cell
                    Cell r = new Cell(x);   // Optimistically create
                    if (cellsBusy == 0 && casCellsBusy()) {
                        boolean created = false;
                        try {               // Recheck under lock
                            Cell[] rs; int m, j;
                            if ((rs = cells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                rs[j] = r;
                                created = true;
                            }
                        } finally {
                            cellsBusy = 0;
                        }
                        if (created)
                            break;
                        continue;           // Slot is now non-empty
                    }
                }
                collide = false;
            }
            else if (!wasUncontended)       // CAS already known to fail
                wasUncontended = true;      // Continue after rehash
            else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                         fn.applyAsLong(v, x))))
                break;
            else if (n >= NCPU || cells != as)
                collide = false;            // At max size or stale
            else if (!collide)
                collide = true;
            else if (cellsBusy == 0 && casCellsBusy()) {
                try {
                    if (cells == as) {      // Expand table unless stale
                        Cell[] rs = new Cell[n << 1];
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        cells = rs;
                    }
                } finally {
                    cellsBusy = 0;
                }
                collide = false;
                continue;                   // Retry with expanded table
            }
            h = advanceProbe(h);
        }
        else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
            boolean init = false;
            try {                           // Initialize table
                if (cells == as) {
                    Cell[] rs = new Cell[2];
                    rs[h & 1] = new Cell(x);
                    cells = rs;
                    init = true;
                }
            } finally {
                cellsBusy = 0;
            }
            if (init)
                break;
        }
        else if (casBase(v = base, ((fn == null) ? v + x :
                                    fn.applyAsLong(v, x))))
            break;                          // Fall back on using base
    }
}
```

讲一下这部分代码的几个重要的点：

- **Line 4**：获取当前线程的 threadLocalRandomProbe 作为 hash 值，如果该值为 0，说明当前线程是第一次进入该方法，这说明该线程之前还没有在 cells 中竞争，本次执行 base 的 CAS 累加失败了，开始竞争 cells 中的元素。注意当参与过 cell 争用之后，线程的 threadLocalRandomProbe 就不再为 0.

---
ThreadLocalRandom.current() 会去判断是否进行初始化：

```java
public static ThreadLocalRandom current() {
    if (UNSAFE.getInt(Thread.currentThread(), PROBE) == 0)
        localInit();
    return instance;
}
```

在 current 方法中判断如果 probe 的值为 0，则执行 locaInit 方法，将当前线程的 probe 设置为非 0 的值，该方法实现如下：

```java
static final void localInit() {
    //private static final AtomicInteger probeGenerator =
    new AtomicInteger();
    //private static final int PROBE_INCREMENT = 0x9e3779b9;
    int p = probeGenerator.addAndGet(PROBE_INCREMENT);
    //prob不能为0
    int probe = (p == 0) ? 1 : p; // skip 0
    long seed = mix64(seeder.getAndAdd(SEEDER_INCREMENT));
    //获取当前线程
    Thread t = Thread.currentThread();
    UNSAFE.putLong(t, SEED, seed);
    //将probe的值更新为probeGenerator的值
    UNSAFE.putInt(t, PROBE, probe);
}
```

probeGenerator 是 static 类型的 AtomicInteger 类，每执行一次 localInit 方法，都会将 probeGenerator 累加一次 0x9e3779b9 这个值，0x9e3779b9这个数字的得来是 2^32 除以一个常数，这个常数就是传说中的黄金比例 1.6180339887；然后将当前线程的 threadLocalRandomProbe 设置为 probeGenerator 的值，如果 probeGenerator 为 0，则取 1

---

- **Line 10**：无限循环代表不停的自旋操作，只有累加成功才会退出。
- **Line 13**：这里使用 `(n - 1) & h` 来得到 cells 数组中的位置，其实是等价于 `h % n`，也就是取模操作，也就是使用哈希值对数组的长度取模，得到值所在的索引位置。

---
位运算(&)效率要比取模运算(%)高很多，主要原因是位运算直接对内存数据进行操作，不需要转成十进制，因此处理速度非常快。

```java
a % b == a & (b - 1)
```

**前提：b 为 2^n**

这块的原理见：[【Java】使用位运算(&)代替取模运算(%)](https://juejin.im/entry/5bbf10c5e51d450e67499afb)

---

## LongAdder 的实现
LongAdder 继承自 Striped64，它的方法只针对简单的情况：cell 存在且更新无竞争，其余情况都通过 Striped64 的 longAccumulate 方法来完成。

```java
public void add(long x) {
     Cell[] as; long b, v; int m; Cell a;
     if ((as = cells) != null || !casBase(b = base, b + x)) {
          // cells 不为空 或在 base 上cas失败。也即出现了竞争。
          boolean uncontended = true;

          if (as == null || (m = as.length - 1) < 0 ||
               (a = as[getProbe() & m]) == null ||
               !(uncontended = a.cas(v = a.value, v + x)))
               // 如果所映射的槽不为空，且成功更新则返回，否则进入复杂处理流程。
               longAccumulate(x, null, uncontended);
     }
}

// 获取当前的和。base值加上每个cell的值。
public long sum() {
     Cell[] as = cells; Cell a;
     long sum = base;
     if (as != null) {
          for (int i = 0; i < as.length; ++i) {
               if ((a = as[i]) != null)
                    sum += a.value;
          }
     }
     return sum;
}
```

注意 sum 方法是非线程安全的，在并发情况下 sum 的值不一定准确，因为当在 sum 的过程中，有可能别的线程正在操作 cells（因为没有加锁）。所以如果要求严格准确的话，需要我们自己加锁。

## 总结
LongAdder 就是基于 Striped64 实现，用于并发计数时，若不存在竞争或竞争比较低时，LongAdder 具有和 AtomicLong 差不多的效率。但是，高并发环境下，竞争比较严重时，LongAdder 的 cells 表发挥作用，将并发更新分散到不同 Cell 进行，有效降低了 CAS 更新的竞争，从而极大提高了 LongAdder 的并发计数能力。因此，高并发场景下，要实现吞吐量更高的计数器，推荐使用 LongAdder。
