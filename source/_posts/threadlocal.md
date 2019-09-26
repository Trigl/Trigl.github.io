---
layout:     post
title:      "ThreadLocal 详解"
date:       2019-09-24
author:     "Ink Bai"
catalog:    true
header-style: "text"
tags:
    - JDK 源码
    - 并发
---
> 今天来深入研究一下 JDK 中的 `ThreadLocal` 类。ThreadLocal 由 Java 界的两个大师级的作者编写，Josh Bloch 和 Doug Lea。Josh Bloch 是 JDK5 语言增强、Java集合(Collection)框架的创办人以及《Effective Java》系列的作者。Doug Lea是 JUC(java.util.concurrent) 包的作者，Java 并发编程的泰斗。所以，ThreadLocal 的源码十分值得学习。

## 为什么要使用 ThreadLocal？

## 一个栗子
ThreadLocal 类提供了 thread-local 类型变量，这种变量与普通的通过 `get` 或者 `set` 方法访问的变量相比，每个线程会对应一个独立的变量值，**它是与线程相关的一个类的私有静态变量**，例如一个用户 ID 或事务 ID 就会被设置成一个 ThreadLocal 类型的变量。

例如下面这个例子，通过 ThreadLocal 变量可以给每个线程指定 ID。

```java
public class ThreadId {
    // ID 生成器
    private static final AtomicInteger nextId = new AtomicInteger(0);

    // 包含线程 ID 的 ThreadLocal 变量
    private static final ThreadLocal<Integer> threadId =
            ThreadLocal.withInitial(() -> nextId.getAndIncrement());
    // 另一种创建 ThreadLocal 实例的实现：new 一个匿名内部类，当然有了 lambda 表达式就不要用这种方式了
//    private static final ThreadLocal<Integer> threadId = new ThreadLocal<Integer>() {
//        @Override
//        protected Integer initialValue() {
//            return nextId.getAndIncrement();
//        }
//    };

    // 获取当前线程 ID
    public static int get() {
        return threadId.get();
    }

    public static void main(String[] args) {
        for (int i = 0; i < 5; i++) {
            Thread t = new Thread(() -> {
                System.out.println("线程:" + Thread.currentThread().getName() + ", ID:" + ThreadId.get());
            });

            t.start();
        }
    }
}
```

为什么需要 ThreadLocal 这种线程私有的实例呢？

在 Java 中，避免并发问题最简单有效的方法就是不要引入并发，也就是多个线程之间不共享变量，也就不会存在并发问题。

当**线程存活**并且 ThreadLocal 实例可以被访问时，每个线程都会有一个该 ThreadLocal 变量引用，这些引用都不相同，可以看作是该 ThreadLocal 变量的多个副本。一旦线程不存在，这些副本就被 GC 掉了。

看完例子，再来看一下 ThreadLocal 提供的方法：

![](/img/content/public.png)

## 初始化方法：`withInitial` & `initialValue`

首先来看一下创建 ThreadLocal 对象的方法，API 提供了两种方法，一种是直接通过构造函数创建，另一种是通过 `withInitial` 方法，这里我们先看一下 `withInitial` 方法的具体实现，因为其使用到了 Java 8 的函数式编程接口。

```java
public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
    return new SuppliedThreadLocal<>(supplier);
}
```

可以看到这里的实现是通过给定一个 `Supplier` 类型的参数，new 了一个 `SuppliedThreadLocal` 的内部类，`Supplier` 是其构造函数的一个参数，所以关键是搞懂 `SuppliedThreadLocal`。

`SuppliedThreadLocal` 继承自 `ThreadLocal` 类，定义这样一个内部类的作用其实就是为了传入 Java8 的 Lambda 表达式，它的实现如下：

```java
static final class SuppliedThreadLocal<T> extends ThreadLocal<T> {

    private final Supplier<? extends T> supplier;

    SuppliedThreadLocal(Supplier<? extends T> supplier) {
        this.supplier = Objects.requireNonNull(supplier);
    }

    @Override
    protected T initialValue() {
        return supplier.get();
    }
}
```

这个内部类 override 了其父类 `ThreadLocal` 的 `initialValue` 方法，这个方法就是 `ThreadLocal` 实例拿到初始化值时必然会调用到的一个方法：

```java
protected T initialValue() {
    return null;
}
```

这个方法的用法如下：

1. 该方法返回当前 `ThreadLocal` 实例的初始化值
2. 当一个线程第一次访问 `ThreadLocal.get()` 方法式，该方法将被调用
3. 该方法在一种情况下不会被调用，那就是线程访问 `ThreadLocal.get()` 之前先调用了 `set` 方法
4. 该方法在大部分情况下最多会被调用一次，除了一种情况：线程先调用 `ThreadLocal.remove()`，然后调用 `ThreadLocal.get()`
5. 开发者如何使用这个方法呢？一种就是实现 `ThreadLocal` 的一个子类，子类里面重载这个方法；另一种就是 new 一个匿名内部类，直接得到其匿名内部类的实例，正如上面[示例](#一个栗子)注释掉的部分。


回到 `SuppliedThreadLocal`，这里其实就是继承了 `ThreadLocal` 然后重载了 `initialValue()`。重载以后的值就是 `supplier.get()`，让我们看一下 `Supplier` 到底是个什么东东：

```java
@FunctionalInterface
public interface Supplier<T> {

    /**
     * Gets a result.
     *
     * @return a result
     */
    T get();
}
```

这个接口加上了 `@FunctionalInterface` 注解，是一个典型的函数式编程接口，关于函数式编程接口可以专门再开一篇来讲了，这里可以通过 [函数式接口 Functional Interface](https://www.cnblogs.com/chenpi/p/5890144.html) 简单了解一下，总之，这里 `Supplier` 的作用其实就是一个我们输入 Lambda 函数式参数，然后它提供给我们需要的对象（T）。

注意在 `java.util.function` 包下面有很多类似于 `Supplier` 的类，可以帮助我们快乐的使用函数式编程接口：

|接口|参数类型|描述|
|-|-|-|
|Consumer|Consumer<T>|接收 T 对象，不返回值|
|Predicate|Predicate<T>|接收 T 对象并返回 boolean|
|Function|Function<T, R>|接收 T 对象，返回 R 对象|
|Supplier|Supplier<T>|提供 T 对象，不接收值|

## 静态内部类 `ThreadLocalMap` 的实现
当创建了一个 ThreadLocal 对象以后，我们可以通过调用其 `get()` 方法得到此线程本地变量的副本值。

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

可以看到这里的实现是基于 ThreadLocal 静态内部类 ThreadLocalMap，让我们看一下它的实现。
#### 基本数据结构
```java
static class ThreadLocalMap {
  // 内部存储 KV 的基本数据结构，注意 key 就是 WeakReference<ThreadLocal<?>>
  static class Entry extends WeakReference<ThreadLocal<?>> {
      // 真正存放 thread-local 值的地方
      Object value;

      Entry(ThreadLocal<?> k, Object v) {
          super(k);
          value = v;
      }
  }

  // hash 表，会进行自动扩容，但是其长度必须是 2 的平方
  private Entry[] table;

  // 哈希表初始容量，必须是 2 的平方
  private static final int INITIAL_CAPACITY = 16;

  // hash 表中元素（Entry）的数量
  private int size = 0;

  // resize 的阈值，默认为 0
  private int threshold;

  // 设置下一次需要扩容的阈值，设置值为输入值len的三分之二
  private void setThreshold(int len) {
      threshold = len * 2 / 3;
  }

  // 以len为模增加i
  // 可以看作环形数组的下一个索引
  private static int nextIndex(int i, int len) {
      return ((i + 1 < len) ? i + 1 : 0);
  }

  // 以len为模减少i，环形数组的上一个数组
  private static int prevIndex(int i, int len) {
      return ((i - 1 >= 0) ? i - 1 : len - 1);
  }
}
```

ThreadLocalMap 是专用于维护线程本地变量的 hash map，这个类定义成了包私有的，这样是为了在 Thread 类中声明，从而在线程中维护一个 ThreadLocalMap 的引用。

```java
ThreadLocal.ThreadLocalMap threadLocals = null;
```

这里注意到十分重要的一点：ThreadLocalMap$Entry 是 WeakReference(弱引用)，并且键值 Key 为 ThreadLocal<?> 实例本身，这里使用了无限定的泛型通配符。

![](/img/content/threadlocalmap.png)
![](/img/content/threadlocalmap-detail.png)

看一下 ThreadLocalMap 的构造函数，是惰性加载的，即只有当有元素要存放的时候才会构建。

```java
ThreadLocalMap(java.lang.ThreadLocal<?> firstKey, Object firstValue) {
    // 初始化table数组
    table = new Entry[INITIAL_CAPACITY];
    // 用firstKey的threadLocalHashCode与初始大小16取模得到哈希值
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    // 初始化该节点
    table[i] = new Entry(firstKey, firstValue);
    // 设置节点表大小为1
    size = 1;
    // 设定扩容阈值
    setThreshold(INITIAL_CAPACITY);
}
```

#### 巧妙的取模操作
上面代码有一行是 ThreadLocalMap 的哈希算法，哈希算法就是根据 key 得到对应的 hash 值：

```java
int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
```

这里的位运算的实质是去一个取模（求余）运算，决定一个 key 应该放在数组的哪个 index 上。

当取模运算中，除数是 2 的 N 次方时，既这个数用二进制表示的时候一定只有一个 1，比如 16，在 Java 的 Integer 中的二进制形式实质就是：

`000000000000000000000000000010000`

减 1 就是：

`000000000000000000000000000001111`

与被除数做与运算，被除数刚好高位就被消除，只剩下低位。即比除数大，但没有超过一倍的部分被保留。这刚好是取模（求余）运算。

之所以这么做，是因为位运算的效率要远高于普通的取模运算。

#### 为什么要用 `0x61c88647`
看完了取模操作，再看一下 `firstKey.threadLocalHashCode` 的具体实现：

```java
// 每一个 ThreadLocal 实例对应的哈希值是不可变的
private final int threadLocalHashCode = nextHashCode();

private static AtomicInteger nextHashCode = new AtomicInteger();

// hash 函数递增的魔法值
private static final int HASH_INCREMENT = 0x61c88647;

private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```

首先要注意到一点，threadLocalHashCode 是一个 final 的属性，而原子计数器变量 nextHashCode 和生成下一个哈希魔数的方法 nextHashCode() 是静态变量和静态方法，静态变量只会初始化一次。换而言之，每新建一个 ThreadLocal 实例，它内部的 threadLocalHashCode 就会增加 0x61c88647。举个例子：

```java
//t1中的threadLocalHashCode变量为0x61c88647
ThreadLocal t1 = new ThreadLocal();
//t2中的threadLocalHashCode变量为0x61c88647 + 0x61c88647
ThreadLocal t2 = new ThreadLocal();
//t3中的threadLocalHashCode变量为0x61c88647 + 0x61c88647 + 0x61c88647
ThreadLocal t3 = new ThreadLocal();
```

threadLocalHashCode 是 ThreadLocalMap 结构中使用的哈希算法的核心变量，对于每个 ThreadLocal 实例，它的 threadLocalHashCode 是唯一的。

这里有一个 hash 递增的魔法值 `0x61c88647`，为什么选这样一个值呢？想要弄明白这一点首先复习一下斐波那契数列：

> 斐波那契数列：1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, …
通项公式：F(n)=F(n-1)+F(n-2)

有趣的一点是，当 n 趋向于无穷大时，前一项与后一项的比值越来越逼近 0.618（或者说后一项与前一项的比值小数部分越来越逼近 0.618），而这个值 0.618 就被称为黄金分割数。证明过程如下：

![](/img/content/fibonaqi.jpeg)

黄金分割数的准确值为(根号5 - 1)/2，约等于 0.618。

黄金分割数被广泛使用在美术、摄影等艺术领域，因为它具有严格的比例性、艺术性、和谐性，蕴藏着丰富的美学价值，能够激发人的美感。当然，这些不是本文研究的方向，我们先尝试求出无符号整型和带符号整型的黄金分割数的具体值：

```java
public static void main(String[] args) {
    // 黄金分割数 * 2的32次方 = 2654435769 - 这个是无符号32位整数的黄金分割数对应的那个值
    long c = (long) ((1L << 32) * (Math.sqrt(5) - 1) / 2);
    System.out.println("32 位无符号整型的黄金分割数：" + c);
    // 强制转换为带符号为的32位整型，值为-1640531527
    int i = (int) c;
    System.out.println("32 位有符号整型的黄金分割数：" + i);
}
```

结果如下：

```java
32 位无符号整型的黄金分割数：2654435769
32 位有符号整型的黄金分割数：-1640531527
```

上面有一个 long 类型强转 int 类型的操作，最后得到的是一个负数。在 Java 中对 int 的越界处理是这样的：当一个数超过了 `Integer.MAX_VALUE` 后，Java 就会从 Integer 的另一头重新开始，也就是从 `Integer.MIN_VALUE` 往回倒推，所以最终越界数显示的结果就是 `Integer.MIN_VALUE + (越界数 - Integer.MAX_VALUE) - 1`。

而 ThreadLocal 的哈希魔数正是 32 位有符号整型黄金分割数 `1640531527` 的十六进制 `0x61c88647`，通过相关理论研究和实践证明发现，使用这个魔数可以使对应的 key 经过 hash 算法后均匀分布到整个容器，可以实现了完美散列。

#### ThreadLocalMap 为什么使用 WeakReference 作为其 key 的类型？

源码中是这样解释的：

> To help deal with very large and long-lived usages, the hash table entries use WeakReferences for keys.
为了应对大量并且长期地使用 ThreadLocal，哈希表使用了弱应用作为其 key 的类型。

大量使用意味着对应的 key 的数目会很多，而长期使用则是由于 ThreadLocal 的生命周期和线程的生命周期一样长。

如果这里使用普通的 key-value 形式来定义存储结构，实质上就会造成节点的生命周期与线程强绑定，只要线程没有销毁，那么节点在 GC 分析中一直处于可达状态，没办法被回收，而程序本身也无法判断是否可以清理节点。弱引用是 Java 中四档引用的第三档，比软引用更加弱一些，如果一个对象没有强引用链可达，那么一般活不过下一次 GC。当某个 ThreadLocal 已经没有强引用可达，则随着它被垃圾回收，在 ThreadLocalMap 里对应的 Entry 的键值会失效，这为 ThreadLocalMap 本身的垃圾清理提供了便利。

看起来结局皆大欢喜，`引用 ThreadLocal 的对象被回收` -> `ThreadLocalMap 弱应用 key 被回收`，真的没问题了吗？既然是一个 map，那就说明数据结构是 key-value 对，现在仅仅 K 使用了弱引用然后被回收了，那么 value 呢？value 为什么不使用弱引用类型？过期的 value 会被回收吗？

事实上当 ThrealLocalMap 的 key 被回收之后，对应的 value 会在下一次调用 `set`、`get`、`remove` 的时候被清除掉。

## `set` & `get`
set 和 get 方法的原理很简单，都是基于 ThreadLocalMap 做的，源码如下：

```java
public void set(T value) {
    // 获取当前线程
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    // map 不为空时直接设置 value，为空时创建新的带初始值的 map
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

public T get() {
    // 得到当前线程
    Thread t = Thread.currentThread();
    // 根据线程得到对应的 ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        // 获取 Entry
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            // entry 不为空得到对应的 value
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    // 没有找到对应的 value 时进行初始化操作
    return setInitialValue();
}
```

#### ThreadLocalMap 中 set 的实现
ThreadLocalMap 使用哈希表存储 key，哈希表的优点是查找和插入速度比较快，但是缺点是不同的 key 经过哈希函数计算以后得到的数组下标可能存在冲突。那么如何解决冲突呢，ThreadLocalMap 使用的方法是线性探测法，即当 key 经过哈希函数计算得到一个下标，但是却发现该下标的位置已经有元素了，那么就继续找下一个位置，直到找到一个符合条件的位置。

set 方法的主要流程是：

- 首先通过 hash 函数找到存放的数组下标
- 利用线性探测法逐步增加数组下标，比对当前要 set 的 key 和当前下标所在位置上 Entry 的 key 是否相同
- 在上面探测过程中，如果对应位置的 Entry 的 key 恰好是当前要更新的 key，直接更新即可；如果 key 为 null，说明这是一个过期 key，就调用 replaceStaleEntry 方法进行替换更新
- 当探测到某个位置，这个位置没有元素，即 Entry 为空，说明要 set 的 key 是一个新 key，之前没有 set 过，所以可以设置在当前位置
- 最后一步是判断是否有必要扩容

```java
// 基于 ThreadLocal 作为 key，对当前的哈希表设置值，此方法由 ThreadLocal.set() 调用
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    // 在 table 中的下标
    int i = key.threadLocalHashCode & (len-1);

    //在i的基础上，不断向前探测，即线性探测法。探查是否已经存在相应的key，如果存在就替换。
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        // 获取Entry持有的ThreadLocal弱引用
        ThreadLocal<?> k = e.get();

        // 如果两个ThreadLocal相等，表示需要更新该key对应的value值
        if (k == key) {
            e.value = value;
            return;
        }

        // 如果k为null，表示Entry持有的弱引用已经过期，即ThreadLocal对象被GC回收了
        if (k == null) {
            // 此时更新旧的Entry值
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    // 下标 i 所在的位置没有元素，说明设置的这个值是新的元素
    tab[i] = new Entry(key, value);
    int sz = ++size;
    // 当ThreadLocalMap达到了一定的容量时，需要进行扩容
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

上面 set 方法代码中有两个方法比较关键，分别是 `cleanSomeSlots` 和 `replaceStaleEntry`，依次来看一下：

```java
/**
 * 启发式地对Entry[]进行扫描，并清理无效的slot.
 * 从下面的while循环表达式可以知道，第一次扫描的单元是i ~ i+log2(n),
 * 如果在这期间发现了无效slot,那么把n变大到数组的长度，此时扫描单元数为log2(length)。
 * 即，在扫描的期间，如果发现了无效slot，就不断增大扫描范围。因此称之为启发式扫描。
 *
 * @param i 无效slot所在的位置
 * @param n 控制扫描的数组单元数的参数
 */
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        //线性探测向前环形扫描
        i = nextIndex(i, len);
        Entry e = tab[i];
        //如果找到一个无效Entry(Key被回收)
        if (e != null && e.get() == null) {
            //设置n为Entry[]的长度，以增加扫描单元数
            n = len;            
            removed = true;
            //调用清理函数，i就是下一次向前探测的初始位置，
            //因为在[旧i,新i]之间的无效slot都被清理了
            i = expungeStaleEntry(i);      
        }
    // n >>>= 1 表示 n = n >>> 1,>>>表示无符号右移
    } while ( (n >>>= 1) != 0);
    return removed;
}

//这里传入的staleSlot表示这个下标位置的Entry是失效的
private int expungeStaleEntry(int staleSlot) {  
    Entry[] tab = table;
    int len = tab.length;

    //把当前位置Entry的value值置空，同时也把Entry[staleSlot]置空，便于GC回收
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    //线性探测法进行环形探测，回收失效的key值及Entry，对于没失效的Entry进行ReHash得到h，
    //再把该Entry放到对h线性探测的下一个为空的位置
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
        (e = tab[i]) != null;
        i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;

                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}

// 在已知存在失效slot的情况下，插入一个key-value值。
// 该方法会触发启发式扫描，清理失效slot。
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                               int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    //向前扫描，寻找一个失效的slot，直到数组元素为null
    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len))
        if (e.get() == null)
            slotToExpunge = i;

    //向后扫描，直到找到一个key与参数的key相等的位置，
    //或者遇到数组元素为null停止扫描
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        //如果找到相同的key，把i位置的Entry与staleSlot位置的Entry交换位置
        //经过这一步骤，失效的slot被移到了i位置
        if (k == key) {
            e.value = value;

            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            //从slotToExpunge位置开始启发式清理过程，该位置根据在前向扫描过程中
            //是否找到另一个失效slot来决定，如果找到，则从该位置开始清理；
            //否则，从i位置开始清理，即上面被交换了位置的slot。
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        //如果前向扫描没有失效slot，并且在后向扫描的过程中遇到了第一个失效slot，记录下该位置
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    //在staleSlot位置插入新值
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

    //从失效slot位置进行启发式清理
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

#### ThreadLocalMap 的 getEntry 的实现
再来看一下 ThreadLocalMap 的 getEntry 方法的源码：

```java
private Entry getEntry(ThreadLocal<?> key) {
    //通过散列函数计算数组下标
    int i = key.threadLocalHashCode & (table.length - 1);   
    Entry e = table[i];
    //如果直接命中，则返回
    if (e != null && e.get() == key)                        
        return e;
    else
        return getEntryAfterMiss(key, i, e);                
}

private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    //利用线性探测法来寻找key所在的位置  
    while (e != null) {                
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        //如果当前遍历到的key已经被回收了，那么进行清理
        if (k == null)                 
            expungeStaleEntry(i);
        else
            //利用环形数组的原理来变化i值
            i = nextIndex(i, len);     
        e = tab[i];
    }
    return null;
}
```

逻辑还是很清晰的，如果通过散列函数得到的数组下标直接命中key值，那么可以直接返回，否则进一步调用getEntryAfterMiss(key,e)方法来进行线性探测查找key。
值得注意的是，这里把查找过程分成了两个方法来处理，为什么要这样做？从源码的注释可以看出，这样设计的目的是最大限度提高getEntry(key)方法的性能，也即是提高直接命中时的返回结果的效率。这是因为JVM在运行的过程中，如果一些短函数被频繁的调用，那么JVM会把它优化成内联函数，即直接把函数的代码融合进调用方的代码里面，这样省掉了函数的调用过程，效率也会得到提高。

## ThreadLocal 的最佳实践
最佳实践很简单，就是每次使用完ThreadLocal实例，都调用它的remove()方法，清除Entry中的数据。

调用remove()方法最佳时机是线程运行结束之前的finally代码块中调用，这样能完全避免操作不当导致的内存泄漏，这种主动清理的方式比惰性删除有效。
## `InheritableThreadLocal` 类
InheritableThreadLocal 继承自 ThreadLocal，它的作用就是当存在子线程的时候，把父线程的 thread-local 变量传递给子线程，当然，传递这些 thread-local 变量实际上传递的也就是 ThreadLocalMap，下面是创建 InheritableThreadLocal 对象之前需要调用的 ThreadLocalMap 的私有构造函数，可以发现就是拷贝了一份父 ThreadLocalMap：

```java
/**
 * Construct a new map including all Inheritable ThreadLocals
 * from given parent map. Called only by createInheritedMap.
 *
 * @param parentMap the map associated with parent thread.
 */
private ThreadLocalMap(ThreadLocalMap parentMap) {
    Entry[] parentTable = parentMap.table;
    int len = parentTable.length;
    setThreshold(len);
    table = new Entry[len];

    for (int j = 0; j < len; j++) {
        Entry e = parentTable[j];
        if (e != null) {
            @SuppressWarnings("unchecked")
            ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
            if (key != null) {
                Object value = key.childValue(e.value);
                Entry c = new Entry(key, value);
                int h = key.threadLocalHashCode & (len - 1);
                while (table[h] != null)
                    h = nextIndex(h, len);
                table[h] = c;
                size++;
            }
        }
    }
}
```

从上面的代码可以发现，子类 thread-local 的值是通过 `childValue` 这个方法获得的，一般父线程和子线程 thread-local 的值是相等的，InheritableThreadLocal 将这个方法声明为 `protected` 的，因此如果我们希望子线程的 thread-local 值和父线程不一样，就可以 override 这个方法：

```java
protected T childValue(T parentValue) {
    return parentValue;
}
```

## Refer
[When and how should I use a ThreadLocal variable?](https://stackoverflow.com/questions/817856/when-and-how-should-i-use-a-threadlocal-variable)
[函数式接口 Functional Interface](https://www.cnblogs.com/chenpi/p/5890144.html)
[为什么Java内部类要设计成静态和非静态两种？](https://www.zhihu.com/question/28197253/answer/365692360)
[ThreadLocal源码分析-黄金分割数的使用](https://www.throwable.club/2019/02/17/java-currency-threadlocal/#%E9%BB%84%E9%87%91%E5%88%86%E5%89%B2%E6%95%B0%E7%9A%84%E5%BA%94%E7%94%A8)
[【细谈Java并发】谈谈ThreadLocal](https://benjaminwhx.com/2018/04/28/%E3%80%90%E7%BB%86%E8%B0%88Java%E5%B9%B6%E5%8F%91%E3%80%91%E8%B0%88%E8%B0%88ThreadLocal/)
[Java基础-ThreadLocal中的中哈希算法0x61c88647](https://www.jianshu.com/p/bf7cdb8e379c)
[How does Java handle integer underflows and overflows and how would you check for it?](https://stackoverflow.com/questions/3001836/how-does-java-handle-integer-underflows-and-overflows-and-how-would-you-check-fo)
[ThreadLocal源码解读](https://www.cnblogs.com/micrari/p/6790229.html)
