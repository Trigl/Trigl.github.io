---
layout:     post
title:      "Scala Implicit 详解"
date:       2018-02-07
author:     "Ink Bai"
catalog:    true
header-img: "/img/post/post-bg-implicit.jpg"
tags:
    - Scala
---

> Implicit 是 Scala 中一个很重要的特性，开始学习 Scala 之前一直以为它和 Java 差不多，然而真的看一些 Scala 的源码时却发现并没有想象中那么简单，所以准备写几篇文章来详解 Scala 中异于 Java 的特性，就从 Implicit 开始吧。

在我看来，Implicit 做的事情也是 Scala 主要做的事情，那就是代码压缩，减少模块代码，talk is cheap，先用一个实例来了解一下 Implicit 的作用。

## 一个栗子
马上就到情人节了，你会如何表达你的爱意呢？

让我们做一些准备工作，首先要有一个恋人的接口，包含一个表达爱意的函数`sendLove`

```scala
trait Lover {
  def sendLove(love: Love) {}
}
```

然后再来一个爱意接口，爱不能总是在心口难开，而是应当有所行动，所以有一个行动函数`takeAction`，`event`是触发爱的事件，这里当然是情人节这件大事了

```scala
trait Love {
  val event: Event

  val show = takeAction(event)

  def takeAction(event: Event)
}
```

今年的情人节很尴尬，正好在过年的前一天，所以这一天很多恋人可能都各呆各家各找各妈了，所以这里应当是异地恋，吼吼吼吼让你们虐狗

```scala
class RemoteLover extends Lover {}

val lover: Lover = new RemoteLover
```

OK，准备工作完成，情人节到了，异地恋们来表达一波爱意吧！

```scala
// without implicit
lover.sendLove(
  new Love {
    val event = Event("Valentine is coming!")

    def takeAction(event: Event) = {
      println("Buy a gift!")
    }
  }
)
```

爱你的方式多种多样，但爱你的心却永远不变。可能是送一件礼物聊表心意，也可能是飞奔到你身边给一个拥抱。我并不是在装情圣，我只是想说明，上面代码中，会改变的其实只有这行代码

```scala
println("Buy a gift!")
```

`Love`及其实现的函数`takeAction`可以说是固定的模块化代码，如果能缩减成下面这个样子那就太棒了

```scala
lover.sendLove(
  (_: Event) => println("Give a hug!")
)
```

看一下发生了什么。函数`sendLove`所需要的参数是`Love`对象：

```scala
new Love {
  val event = Event("Valentine is coming!")

  def takeAction(event: Event) = {
    println("Buy a gift!")
  }
}
```

但是缩减后其对应的参数变成了一个函数：

```scala
(_: Event) => println("Give a hug!")
```

不行啊！这样搞参数类型不匹配啊！这时候就该想到类型转换了。如下是一个超简单的例子：

```scala
"abc" + 123
```

这里 123 是 Int 转换成了 String，参考这个思路，试着将函数转化成`Love`：

```scala
implicit def function2Love(f: Event => Unit) = new Love {
  val event = Event("Valentine is coming!")

  def takeAction(event: Event): Unit = f(event)
}
```

这时原来的代码就可以写成：

```scala
lover.sendLove(
  function2Love(
    (_: Event) => println("Give a hug!")
  )
)
```

这里我们明确指定函数`function2Love`进行类型转换，注意上面`function2Love`的前面使用了 implicit 关键字，这个关键字的作用就是可以隐式将一种类型转换为另一种类型，而不需要明确调用，如下：

```scala
// with implicit
import Lover.function2Love

lover.sendLove(
  (_: Event) => println("Give a hug!")
)
```

在这里具体发生了什么呢？首先`sendLove`需要的参数类型是`Love`,但是这里的参数却是一个函数。当编译器发现代码中存在不匹配的类型的时候，它不会立即报错，而是选择先抢救一下，抢救的方式就是在当前代码域中找一下有没有 implicit 修饰的代码，然后找到了`function2Love`这个隐式转换函数，代码编译通过。

以上就是 Implicit 的一个入门例子，全部代码见 [GitHub](https://github.com/Trigl/scala-learning/tree/master/src/main/scala/ink/baixin/scalalearning/implicits)

一般来说 Implicit 主要用在两个方面：隐式转换和隐式参数，下面分别进行讲解

---

## 隐式转换
#### 隐式转换到某个期望类型
任何时候编译器发现了类型 X，但是却需要类型 Y 的时候，它就会寻找一个可以把 X 转换成 Y 的隐式函数。

例如正常情况下一个 Double 类型是不能转换成 Int 类型，但是我们可以定义一个隐式函数来实现：

```scala
scala> implicit def doubleToInt(x: Double) = x.toInt
doubleToInt: (x: Double)Int
```

虽然我在这里举了一个这样的例子，但是这种从 Double 转到 Int 的方式是不推荐的，因为会丢失精度。相反从 Int 转到 Double 就是正常的，而且其底层实现就是用的隐式转换。Scala 有一个`scala.Predef`对象，默认引入到每一个程序中，就包含了类似这种从“小”数类型向“大”数类型的隐式转换：

```scala
implicit def int2double(x: Int): Double = x.toDouble

scala> val x: Double = 3
x: Double = 3.0
```

#### 隐式类
隐式类是 Scala2.10 新增的，为了更方便地写包装器类。有以下几个要点：

1. 隐式类不能是 case class，并且构造器必须只有一个参数
2. 一个隐式类必须在其他 object，class，trait 的内部
3. 对于一个隐式类，编译器会自动生成一个隐式转换函数，该函数会产生一个隐式类对象

例如一个矩形类 `Rectangle`

```scala
case class Rectangle(width: Int, height: Int)
```

新建一个矩形需要这样：

```scala
scala> val rec = Rectangle(3, 4)
rec: Rectangle = Rectangle(3,4)
```

这还是有点麻烦的，如果我们想通过 `3 x 4` 这种方式就新建一个对象那该怎么办呢，这个时候就要用到隐式类了：

```scala
implicit class RectangleMaker(width: Int) {
  def x(height: Int) = Rectangle(width, height)
}
```

我们再来建立一个矩形：

```scala
scala> val easyRec = 3 x 4
easyRec: Rectangle = Rectangle(3,4)
```

这看起来就简单直接多了嘛，那么它是怎么实现的呢？前面我们说了，对于隐式类，编译器会自动生成一个隐式转换函数，如下：

```scala
// Automatically generated
implicit def RectangleMaker(width: Int) = new RectangleMaker(width)
```

所以当编译器看到 `3 x 4` 的时候，首先发现 3 这个 Int 没有 x 函数，然后就找隐式转换，通过上面的隐式转换函数生成了一个`RectangleMaker(3)`，然后调用`RectangleMaker`的 x 函数，就产生了一个矩形`Rectangle(3,4)`

---

## 隐式参数
下面要开始谈隐式参数了，这也是我们最常用到的，隐式参数并不是用于类型转换的，更多的是为了减少代码量，要点如下：

1. 隐式参数是指类或者函数所对应的参数列表，如下面 implicit 所在的参数列表

  ```scala
  implicit class Greet(name: String)(implicit hello: Hello, world: World)
  ```

2. implicit 放在参数列表的最前面，不管有几个参数，它们全部都是隐式的
3. 定义了隐式参数以后，我们新建一个类或者调用一个函数的时候就可以省略该参数列表了

  ```scala
  val greet = new Greet("Jack")
  ```

4. 省略参数列表并不是不需要参数列表，我们需要使用 implicit 关键字定义出隐式参数列表中的所有变量，然后使用 import 导入

  ```scala
  implicit val hello = new Hello
  implicit val world = new Wrold
  ```

上面就是一些基本点，狗年春节马上就要来了，就让我们以此为例来写一个回家的 Demo 吧

```scala
case class Remote(address: String)
case class Home(address: String)

object Transportation {
  def transport(name: String)(implicit remote: Remote, home: Home) = {
    println(s"To celebrate Spring Festival, go from $remote to $home, by $name")
  }
}

object Address {
  implicit val remote = new Remote("Shanghai")
  implicit val home = new Home("Shanxi")
}
```

这里定义了起始地址`Remote` `Home`，虽然其本质是 String 类型，但是对于隐式参数来说最好还是定义一个专门的类，因为如果直接使用 String 作为隐式变量，由于其太过于普遍编译器可能不太容易找到，或者和其他隐式变量混淆，而专用的类就不用担心这些。
还有一个表示交通方式的函数`transport`，该函数里面用到了隐式参数，最后`Address`对象定义了我们需要的隐式变量。

然后，准备回家喽！

```scala
object GoHome extends App {
  import Address._
  Transportation.transport("airplane")
}
```

结果如下：

```scala
To celebrate Spring Festival, go from Remote(Shanghai) to Home(Shanxi), by airplane.
```

---

## 上下文绑定
了解了隐式参数以后，我们可以再看一个有趣的语法糖：上下文绑定 `context bound`

这里我用 *Programming in Scala* 中的一个例子来讲解，要求找出一个 List 中所有元素的最大值，首先我们看一下常规的实现了隐式参数的写法：

```scala
// with implicit
def maxListOrdering[T](elements: List[T])
                      (implicit ordering: Ordering[T]): T =
  elements match {
    case List() =>
      throw new IllegalArgumentException("empty lists!")
    case List(x) => x
    case x :: rest =>
      val maxRest = maxListOrdering(rest)
      if (ordering.gt(x, maxRest)) x
      else maxRest
  }
```

这个函数的参数有两个，一个是 T 组成的 List，另一个是隐式参数 Ordering，对于一些常见的 T 类型，例如 Int 或者 String，它们都有一个默认的 Ordering 实现，所以不需要定义其隐式变量就可以实现排序：

```scala
println(maxListOrdering(List(1, 5, 10, 3)))
```

在上面代码中有一行用到了 ordering 参数：

```scala
if (ordering.gt(x, maxRest)) x
```

这里 ordering 就是 Ordering[T] 的一个对象，事实上 Scala 标准库中提供了一个方法让编译器可以自己寻找到一个类的隐式变量：

```scala
def implicitly[T](implicit t: T) = t
```

例如一个类型 `Foo`，调用`implicitly[Foo]`会产生什么结果呢？首先编译器会寻找 Foo 的隐式对象，找到以后调用这个对象的 implicitly 方法，然后返回该对象，所以通过`implicitly[Foo]`可以返回 Foo 的隐式对象，对应到我们上面举的例子，ordering 是 Ordering[T] 的一个隐式对象，事实上我们也可以通过 `implicitly[Ordering[T]]`来表示

```scala
if (implicitly[Ordering[T]].gt(x, maxRest)) x
```

那么现在看来，ordering 既然可以用 implicitly[Ordering[T]] 来代替，那么也就可以省略掉这个参数名了，所谓的上下文绑定就是做这个事情的，语法是 `[T: Ordering]`，它主要做了两件事：第一，引入了一个参数类型 T；第二，增加了一个隐式参数 Ordering[T]。
与之很类似的一个语法是`[T <: Ordering[T]]`，这个的意思是 T 就是一个 Ordering[T]。
现在再来看一下引入上下文绑定后的排序代码：

```scala
// context bound
def maxListOrdering[T: Ordering](elements: List[T]): T =
  elements match {
    case List() =>
      throw new IllegalArgumentException("empty lists!")
    case List(x) => x
    case x :: rest =>
      val maxRest = maxListOrdering(rest)
      if (implicitly[Ordering[T]].gt(x, maxRest)) x
      else maxRest
  }
```

---

## 总结
implicit 使用起来还是很灵活的，可以将 implicit 关键字标记在变量、函数、类或者对象的定义中。而且记住想要用到隐式，一定要先使用 import 引入 implicit 代码（当然在同一个代码域例如同一个对象下面不需要）。但是也需要注意，如果太过于频繁地使用 implicit，代码可读性就会很低，所以在使用隐式转换之前，先看一看能否用其他方式例如继承、重载来实现，如果都不行并且代码仍然很冗余，那么就可以试试用 implicit 来解决。
