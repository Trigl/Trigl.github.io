---
layout:     post
title:      "Effective Java 读书笔记：Item 1"
date:       2019-08-25
author:     "Ink Bai"
header-img: "/img/post/effective-java-1.jpg"
tags:
    - Effective Java
---

> 几年前看过 Effective Java 第二版，当时看的是一知半解，就被我放下了，最近发现 Effective Java 都出了第三版了，涵盖到了 Java 9 的内容，准备重新过一遍，顺便撸一点读书笔记吧。

首先来看 Item 1：

> 考虑使用静态工厂方法替换构造方法

通常我们创建对象的方式是调用类的构造函数，其实还有一种更优雅的方式是通过静态工厂方法创建对象，比如下面的 Boolean 类的一个静态工厂方法：

```java
public static Boolean valueOf(boolean b) {
   return b ? Boolean.TRUE : Boolean.FALSE;
}
```

注意静态工厂方法和设计模式中的工厂方法模式并不相同，静态工厂方法相对于构造函数有什么优缺点呢，让我们逐个看一下。

**优点一：静态工厂方法有具体的命名，可读性好**
一个类可能有多个构造方法，每个构造方法的参数列表是不同的，仅仅这些我们并不能很好的区分它们的差别。而静态工厂方法可能就会有一个专门的名称标识生成的是什么类，一目了然，代码更易于阅读。例如 `BigInteger.probablePrime` 这个静态呢工厂方法就会告诉我们，它会创建一个可能是素数的 `BigInteger` 对象。

**优点二：不同于构造方法，静态工厂方法不需要每次调用都创建一个新对象**
静态工厂方法可以使用预先构造好的实例，或者在第一次构造实例时缓存实例，反复分配它们从而避免创建不必要的重复对象。比如上面的 `Boolean.valueOf(boolean)` 方法就从来不创建对象。

**优点三：静态工厂方法可以单独定义在接口上（或者接口的伴生类），可以很方便地创建这个接口的多个子类对象**
这个优点最明显的体现就是 `java.util.Collections` 类，里面有丰富的静态工厂方法可以创建不可修改的集合和同步集合的实现。

![](/img/content/collection-1.jpg)

这样做有什么好处呢？首先，这么多创建子类对象的 API 都放在一个类中了，大大减少了我们学习和使用的难度；其次，可以直接通过接口而不是实现类来调用返回的对象，这通常是一种良好的编程实践。

当然对于集合类框架是通过一个具体的类 `Collections` 而不是通过接口 `Collection` 来创建的子类对象，这是因为 Java 8 之前，接口内是不能包含静态方法的，Java 8 之后才取消了这个限制。Java 集合类框架的实现远远早于 Java 8，因此当时就给接口 `Collection` 设计了一个伴生类 `Collections` 来实现各种静态工厂方法。

**优点四：返回对象的类可以根据输入参数的不同而不同**
如果父类有多个子类的实现，那么可以根据输入参数返回不同的子类对象。例如 `EnumSet` 这个类与普通 Java 集合类相比，对存储 Enum 做了优化。比如我们创建一个空的 enum set：

```java
Set<SampleEnum> s = Collections.synchronizedSet(EnumSet.noneOf(SampleEnum.class));

enum SampleEnum {
    ENUM1,
    ENUM2,
    ENUM3;
}
```

`EnumSet.noneOf(type)` 的底层实现如下：

```java
public static <E extends Enum<E>> EnumSet<E> noneOf(Class<E> elementType) {
    Enum<?>[] universe = getUniverse(elementType);
    if (universe == null)
        throw new ClassCastException(elementType + " not an enum");

    // 枚举类型包含元素少于等于 64 个，创建 RegularEnumSet 对象
    // 否则创建 JumboEnumSet 对象
    if (universe.length <= 64)
        return new RegularEnumSet<>(elementType, universe);
    else
        return new JumboEnumSet<>(elementType, universe);
}
```

可以看到底层实现做了一些优化，可以根据枚举类型的大小返回 `EnumSet` 的两个子类中其中一个的实例，这些都是底层实现的，对外不可知，调用这个 API 的开发者只知道返回的是 `EnumSet` 的一个子类。

**优点五：灵活的静态方法构成了服务提供者框架的基础**
服务提供者框架可以单开一篇讲一下了，先放入 todo list。

上面说了静态工厂的一堆优点，下面看一下静态工厂的缺点。

**缺点一：一个只提供静态工厂方法的类，由于没有公共或受保护的构造方法，因此不能被子类化**
这个缺点呢，变相的鼓励我们使用 API 的时候多使用组合而不是集成，也可以说是因祸得福吧。

**缺点二：很难找到这些静态工厂方法**
每个类都有构造方法，相对比较好找，而找出静态工厂方法就不是很容易了，我们只能根据一些常见命名传统来找了，比如 `of`、`from`、`valueOf`、`newInstance` 等等。

总之，静态工厂方法和公共构造方法都有它们的用途，并且了解它们的相对优点是值得的。通常，静态工厂更可取，因此避免在没有考虑静态工厂的情况下提供公共构造方法。
