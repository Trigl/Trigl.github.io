---
layout:     post
title:      "Scala 的 Case Classes 和 Pattern Matching"
date:       2018-04-13
author:     "Ink Bai"
catalog:    true
header-img: "http://ox2ru2icv.bkt.clouddn.com/image/post/pattern-matching.jpg"
tags:
    - Scala
---

> 本文将讲解 Scala 中无处不在的 case class 和 pattern matching，为什么要放在一起讲呢，因为 case class 一般就是和模式匹配一起使用，习惯了用这套组合拳以后就再也不想写 Java 代码了，use less code to show more!

## Case Class
case class 是指在 class 前面加上 case 关键字，下面是一个例子：

```scala
abstract class Animal
case class Dog(name: String) extends Animal
case class Cat(name: String) extends Animal
```

case class 和普通 class 相比有以下几个不同的地方：

有一个以该类名为方法名的工厂方法用于实例化这个类，这样的好处是直接写该类名就可以新建一个对象，而不需要在前面加上 `new`

```scala
scala> val dog = Dog("Zeus")
dog: Dog = Dog(Zeus)
```

类的参数列表中的变量声明会默认加上 `val`，所以这些参数就是类的成员变量，可以直接调用

```scala
scala> dog.name
res0: String = Zeus
```

编译器会自动在类中添加 `toString`, `hashCode`, `equals` 方法

编译器会添加一个 `copy` 方法用于对你的类进行复制修改

```scala
scala> dog.copy(name = "Simba")
res1: Dog = Dog(Simba)
```

事实上最大的优势还是可以支持 pattern matching

## pattern matching
#### 模式匹配的格式
基本格式如下，`selector` 是指所要匹配的那个变量，下面对应多个可能性，`pattern` 就是具体对应的类别，比如某个 case class，expression，每个 `pattern` 前必须加 `case` 关键字，箭头后面对应相应的表达式

```scala
selector match {
  case pattern1 => expression1
  case pattern2 => expression2
  ...
}
```

下面是一个简单例子

```scala
def behavior(animal: Animal) = animal match {
  case Dog(name) => println("wangwangwang!")
  case Cat(name) => println("miaomiaomiao~")
  case _ =>
}
```

结果：

```scala
scala> behavior(animal)
wangwangwang!
```

注意两点：

1. match 后所接表达式按照其书写顺序进行判断
2. match 后的多个选择中，每次都有且仅有一个会被选择，也就是必须保证所有的可能性都存在，即使添加一个默认选项什么也不做，否则会抛出 `MatchError` 的异常

#### 模式匹配的类型
**通配符匹配**
通配符匹配是指使用 `_` 进行匹配，会匹配到任何对象，一般通配符匹配会用于模式匹配的默认结果，另一个用法就是当你不关心匹配的内容时，可以使用一个 `_` 来代替一个需要命名的变量，示例如下：

```scala
animal match {
  case Dog(_) => println("wangwangwang!")
  case Cat(_) => println("miaomiaomiao~")
  case _ =>
}
```

**常量匹配**
顾名思义，匹配的类型是具体的常量：

```scala
def describe(x: Any) = x match {
  case 5 => "five"
  case true => "truth"
  case "hello" => "hi"
  case Nil => "the empty list"
  case _ => "something else"
}
```

**变量匹配**
变量匹配和通配符匹配一样，可以匹配到任意对象：

```scala
expr match {
  case 0 => "zero"
  case somethingElse => "not zero: " + somethingElse
}
```

但是注意变量只能使用一次，不能出现 `case Dog(name, name)` 这种情况

另外，对变量加上反引号 ` `` ` 就可以使这个变量变成一个常量匹配，例如：

```scala
val pi = math.Pi

expr match {
  case `pi` => "Pi = " + pi
  case _ => "OJBK"
}
```

这里额外讲一下反引号 ` `` ` 在 Scala 中的两个用途，一个是把 Scala 的关键字变成普通标志符来使用，如 ``Thread.`yield`()``；另一个就是上面所示，把小写字母标志符作为常量来用。

**构造器匹配**
构造器匹配应该是我们最常用到的，即可以匹配到某个类以及类里面对应的成员变量，如果该成员变量也是一个类，那么还可以继续深度匹配下去

```scala
animal match {
  case Dog(name) => println(name)
  case _ =>
}
```

**序列匹配**
可以用于匹配列表或者集合

```scala
expr match {
  case List(0, _, _) => println("found it")
  case _ =>
}
```

上面的例子指定了列表的数量是三个，如果想要不指定数量，可以用正则的方式 `_*`

```scala
expr match {
  case List(0, _*) => println("found it")
  case _ =>
}
```

**多元组匹配**

```scala
expr match{
  case (a, b, c) => println("matched " + a + b + c)
  case _ =>
}
```

**类型匹配**
匹配到具体属于那个类型

```scala
x match {
  case s: String => s.length
  case m: Map[_, _] => m.size
  case _ => -1
}
```

但是这里注意一点，类型匹配只能匹配到这个类的类型，无法匹配到这个类的参数的类型，例如下面这个例子：

```scala
def isIntIntMap(x: Any) = x match {
  case m: Map[Int, Int] => true
  case _ => false
}
```

编译器就会发出警告：

```scala
<console>:12: warning: non-variable type argument Int in type pattern scala.collection.immutable.Map[Int,Int] (the underlying of Map[Int,Int]) is unchecked since it is eliminated by erasure
         case m: Map[Int, Int] => true
                 ^
```

这是因为类似于 Java，Scala 的泛型也是通过类型擦除来实现的，所以在运行时类型参数信息是未知的，因此对应到模式匹配，只能匹配到类的类型，但是无法匹配该类的参数的类型。

唯一的例外就是数组，数组的元素存在数组值中，因此可以匹配到其类型

```scala
def isStringArray(x: Any) = x match {
  case a: Array[String] = "yes"
  case _ => "no"
}
```

**变量绑定**
如果我们需要模式匹配中的某些值，并且无法直接获取到，这时可以使用 `@` 把这个值绑定到一个变量，可以在后面的表达式中使用

```scala
animal match {
  case Dog(name, b @ Behavior("run", _)) => b
  case _ =>
}
```

#### 其他注意点
模式匹配并不是都是 `selector match { alternatives }` 的格式，下面的例子其实也是模式匹配

```scala
for ((country, city) <- capitals)
  println("The capital of " + country + " is " + city)
```

另外，如果需要在模式匹配时加上判断，可以在匹配项的后面加上 `if` 判断条件：

```scala
x match {
  // match only positive integers
  case n: Int if 0 < n => ...
  // match only strings starting with letter 'a'
  case s: String if s(0) == 'a' => ...
}
```

## Sealed classes
有时我们可以看到在一个类前面会有 `sealed` 关键字如下：

```scala
sealed abstract class Animal
case class Dog(name: String) extends Animal
case class Cat(name: String) extends Animal
```

这种类有什么特点呢？一般来说我们会给一个父类设置为 `sealed`，其子类必须与其在同一个文件下面。那么这个与模式匹配有什么关系呢？一般模式匹配我们会把所有的可能性都想到，对应的可能性一般是一个父类下的多个子类，当我们把父类设置为 `sealed` 的时候，如果你模式匹配缺少了某些子类时，编译器就会给出一个警报，可以防止我们遗漏掉子类。

## Option
相信写过 Java 代码的都遇到过头疼的 `NullPointerException`，在 Scala 中使用 Option 来解决这个问题。`Option` 类用来指代可选的值，就是这个值可能有也可能为空，当有值的话它的结果就是 `Some(x)` 的形式，当没有值的时候结果就是 `None`，与模式匹配结合起来就能给出正确的结果：

```scala
def show(x: Option[String]) = x match {
  case Some(s) => s
  case None => "?"
}
```

我们这里使用 `Map` 来验证一下，因为 Scala 中 Map 的 get 方法返回的就是 Option 对象

```scala
val capitals = Map("China" -> "Beijing", "Japan" -> "Tykyo")

scala> show(capitals get "China")
res19: String = Beijing

scala> show(capitals get "America")
res20: String = ?
```

## 总结
模式匹配可以替代 Java 中的两件事，一个是 `if` 语句，另一个是 `switch` 语句，所以如果你还困扰于各种 if 判断逻辑中，那就快把模式匹配引入你的代码里面吧。
