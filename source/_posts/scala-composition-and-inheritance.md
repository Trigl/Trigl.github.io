---
layout:     post
title:      "Scala Composition and Inheritance"
subtitle:   "Notes of Programming in Scala"
date:       2018-01-15
author:     "Ink Bai"
catalog:    true
# header-img: "http://ox2ru2icv.bkt.clouddn.com/image/jpg/post-bg-ddia.jpg"
tags:
    - Scala
---

## parameterless methods && empty parentheses

**parameterless methods**

```scala
def width: Int
```

**empty parentheses**

```scala
def width(): Int
```

However, it's still recommended to write the empty parentheses when the invoked method represents more than a property of its receiver object. For instance, empty parentheses are appropriate if the method performs I/O, writes reassignable variables (vars), or reads vars other than the receiver's fields, either directly or indirectly by using mutable objects.

Another way to think about this is if the function you're calling performs an operation, use the parentheses. But if it merely provides access to a property, leave the parentheses off.

---

## Overriding methods and fields
Another difference is that in Scala, fields and methods belong to the same namespace. This makes it possible for a field to override a parameterless method.

```scala
class ArrayElement(conts: Array[String]) extends Element{
  val contents: Array[String] = conts
}
```

in Scala it is forbidden to define a field and method with the same name in the same class.

Java's four namespaces are fields, methods, types, and packages. By contrast, Scala's two namespaces are:

- values (fields, methods, packages, and singleton objects)
- types (class and trait names)

You can use another another way to write codes above to avoid redundancy and repetition:

```scala
class ArrayElement(
  val contents: Array[String]
) extends Element
```

Note that now the contents parameter is prefixed by val. This is a shorthand that defines at the same time a parameter and field with the same name.

---

## Using `override` modifiers
Scala requires such a modifier for all members that override a concrete member in a parent class. The modifier is optional if a member implements an abstract member with the same name.

---

## Using composition and inheritance
If what you're after is primarily code reuse, you should in general prefer composition to inheritance.

One question you can ask yourself about an inheritance relationship is whether it models an is-a relationship.
Another question you can ask is whether clients will want to use the subclass type as a superclass type.

---

## Implementing `above`, `beside`, and `toString`
Putting one element above another means concatenating the two contents values of the elements.

```scala
def above(that: Element): Element =
  new ArrayElement(this.contents + that.contents)
```

The `++` operation concatenates two arrays.
Specifically, arrays in Scala can be converted to instances of a class scala.Seq.

The zip operator picks corresponding elements in its two operands and forms an array of pairs.
For instance, this expression:

```scala
Array(1, 2, 3) zip Array("a", "b")
```

will evaluate to:

```scala
Array((1, "a"), (2, "b"))
```

The implementation of toString makes use of mkString, which is defined for all sequences, including arrays.

---

## Defining a factory object
A factory object contains methods that construct other objects. Clients would then use these factory methods to construct objects, rather than constructing the objects directly with new.

The first task in constructing a factory for layout elements is to choose where the factory methods should be located.
A straightforward solution is to create a companion object of class Element and make this the factory object for layout elements.

```scala
object Element{
  def elem(contents: Array[String]): Element = {
    new ArrayElement(contents)
  }

  def elem(line: String): Element = {
    new LineElement(line)
  }
}
```

In Scala, you can define classes and singleton objects inside other classes and singleton objects. One way to make the Element subclasses private is to place them inside the Element singleton object and declare them private there. The classes will still be accessible to the three elem factory methods, where they are needed.

```scala
object Element{

  private class ArrayElement(val contents: Array[String]) extends Element

  private class LineElement(s: String) extends Element {
    val contents = Array(s)
    override def width = s.length
    override def height = 1
  }

  def elem(contents: Array[String]): Element = {
    new ArrayElement(contents)
  }

  def elem(line: String): Element = {
    new LineElement(line)
  }
}
```
