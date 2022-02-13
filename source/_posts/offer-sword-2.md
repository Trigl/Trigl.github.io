---
layout:     post
title:      "「剑指 Offer」面试题 2：实现 Singleton 模式"
date:       2018-11-23 01:00:00
author:     "Ink Bai"
catalog:    true
header-img: "/img/post/offer-sword-02.jpg"
catalog:    true
tags:
    剑指 Offer
---

> 题目：设计一个类，我们只能生成该类的一个实例。

## 什么是 Singleton？
单例模式的定义很简单：提供一个全局可以访问的唯一变量。

那么为什么需要单例模式呢？对于某些类创建时需要消耗很多资源，例如数据库连接、日志类、配置类等全局都需要的并且占用很多资源的类，我们就希望节能减排多次利用，这个时候就应当采用单例模式。

那么我们如何实现一个类可以被全局访问到并且具有唯一变量呢？首先这个类需要提供一个可以被访问到其唯一实例的方法，其次还要避免可以被多次创建，即我们可以把构造函数设为私有，保证其不能被除自身之外的其他成员所创建。

## 实现
想要实现单例模式，首先要能全局访问，所以必须有一个定义为 `public` 和 `static` 的方法；同时这个实例必须是唯一的，那也就意味着其只能被创建一次，我们可以先这样实现：

**懒汉模式**


```java
public class LazySingletonOne {
  private static LazySingletonOne instance;

  private LazySingletonOne() {}

  public static LazySingletonOne getInstance() {
    if(instance == null) {
      instance = new LazySingletonOne();
    }
    return instance;
  }
}
```

上面的方法是 `public` 和 `static`，保证了每次都会访问到同一个方法；实例是 `private` 的，保证不能在外部通过 `new` 的方法创建实例，而只能通过这里的 `getInstance` 方法创建，看起来完全满足了单例模式的定义，但是想一个问题，在多线程情况下也满足吗？
可以很容易地发现，多线程模式下，如果实例还没有创建，那么可能会同时创建多个实例，所以需要先判断实例是否为空，如果为空的话就先加同步锁然后再创建实例，而且注意给实例加上 `volatile` 关键字保持变量一致性。

**懒汉模式——双重检查锁**

```java
public class LazySingletonTwo {
  private volatile static LazySingletonTwo instance;

  private LazySingletonOne() {}

  public static LazySingletonTwo getInstance() {
    if(instance == null) { // first check
      synchronized (LazySingletonTwo.class) {
        if(instance == null) { // double check
          instance = new LazySingletonTwo();
        }
      }
    }
    return instance;
  }
}
```

上面的方法使用双重检查的方法已经是一个比较好的实现单例模式的方式了，还有一个更简单的实现方法，是 *Effective Java* 里面推荐的单例模式的最佳实现方式，非常简单。

**枚举类模式**

```java
public enum EnumSingleton {
  INSTANCE;
}
```

这种方式在多线程下是安全的，并且可以防止反序列化创建新的对象，有效预防反射攻击。

## Scala 如何实现单例？
Scala 实现单例特别简单，直接定义单例对象即可：

```scala
object ScalaSingleton {}
```

`object` 关键字创建的对象叫做单例对象，Scala 帮我们实现了单例模式，妈妈再也不用担心我不会写单例模式了！

## 总结
单例模式可以说是一个很简单也很基础的模式，但是它其中的思想却很重要，可以举一反三地应用到软件开发的各个方面。惰性加载和唯一单例是为了节约资源使用，保证高性能，比如我们创建连接的时候会从连接池中复用连接，就是同样的道理。另外一个重要的点就是多线程情况，我们任何时候都要多问一下自己会不会有线程安全问题，保证我们的代码有很高的健壮性。
