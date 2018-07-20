---
layout:     post
title:      "那些容易漏掉的 Scala 基础"
subtitle:   "Notes of Scala Cookbook"
date:       2018-07-17
author:     "Ink Bai"
catalog:    true
header-img: "http://ox2ru2icv.bkt.clouddn.com/image/post/scala-basic.jpg"
tags:
    - Scala
---

> 最近重读了一下 Scala Cookbook 来巩固基础，记录一些容易忘记的关键内容。

## Scala 中方法和函数的区别
举个例子吧，下面是一个方法：

```scala
def towLower(c: Char): Char = (c.toByte + 32).toChar
```

下面是与之完全等价的一个函数：

```scala
val toLower = (c: Char) => (c.toByte + 32).toChar
```

## 使用正则
可以通过在一个 String 对象上调用 `.r` 方法来创建一个 `Regex` 对象，然后使用 `findFirstIn` 来寻找第一个匹配的字符串，使用 `findAllIn` 来寻找所有匹配的字符串：

```scala
scala> val numPattern = "[0-9]+".r
scala> val address = "123 Main Street Suite 101"

scala> val match1 = numPattern.findFirstIn(address)
match1: Option[String] = Some(123)

scala> val matches = numPattern.findAllIn(address)
scala> matches.foreach(println)
123
101
```

也可以使用替换方法，`replaceFirst` 或者 `replaceFirstIn` 替换第一个匹配到的，`replaceAll` 或者 `replaceAllIn` 替换所有匹配到的：

```scala
scala> val result = "123".replaceFirst("[0-9]", "x")
result: java.lang.String = x23

scala> val regex = "H".r
scala> val result = regex.replaceFirstIn("Hello world", "J")
result: String = Jello world

scala> val address = "123 Main Street".replaceAll("[0-9]", "x")
address: java.lang.String = xxx Main Street

scala> val regex = "[0-9]".r
scala> val newAddress = regex.replaceAllIn("123 Main Street", "x")
newAddress: String = xxx Main Street
```

还可以将正则应用到模式匹配中：

```scala
scala> val pattern = "([0-9]+) ([A-Za-z]+)".r
scala> val pattern(count, fruit) = "100 Bananas"
count: String = 100
fruit: String = Bananas
```

## 处理 Option 的常规方式

- 调用 `getOrElse` 来获取原始值
- 应用到模式匹配表达式中
- 应用到 `foreach` 循环中

## 如何给一个封闭的类添加方法？
我们可以通过隐式转换给一个封闭的类（如 String）添加一些新的功能，只要在使用的时候使用 `import` 引入即可。

例如我们想要操作一个 String，最直接的方法就是调用一个工具类，这个工具类的方法中参数类型是 String：

```scala
StringUtilities.increment("HAL")
```

但是这样显得有点冗余，我们想要实现 String 类型可以直接调用某个方法，类似：

```scala
"HAL".increment
```

这该怎么实现呢？这就可以通过强大的隐式类来实现这个方法，首先定义一个隐式类和方法：

```scala
object StringUtils {
  implicit class StringImprovements(val s: String) {
    def increment = s.map(c => (c + 1).toChar)
  }
}
```

然后将这个隐式类引入就可以在 String 上直接调用这个方法了：

```scala
object ImplicitClassTest {
  import utils.StringUtils.StringImprovements

  def main(args: Array[String]): Unit = {
    println("HAL".increment)
  }
}
```

神奇吧！那么这个是怎么实现的呢，让我们简单看一下整个过程：

- 编译器发现了一个字符串 `HAL`
- 编译器发现你想要在 String 类型上调用 `increment` 方法
- 编译器发现 String 类型没有这个方法，所以开始尝试寻找一个作用在 String 上的隐式转换
- 由于我们通过 `import` 引入了 `StringImprovements` 这个隐式类，所以编译器发现了这个类，然后将 String 类型转化为 `StringImprovements`，随之调用其 `increment` 方法

但是注意一点：**隐式类必须定义在类、对象或者包对象中**
