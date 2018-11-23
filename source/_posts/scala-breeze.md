---
layout:     post
title:      "Scala 的矩阵运算"
date:       2018-07-24
author:     "Ink Bai"
catalog:    true
header-img: "/img/post/scala-breeze.jpg"
tags:
    - Scala
---

> 本文讲解 Scala 中进行矩阵运算的包 breeze 的一些基础操作。

[Breeze](https://github.com/scalanlp/breeze) 是 Scala 中用于线性代数相关数值运算的开源库。

## 安装
首先我们使用 SBT 导入：

```sbt
libraryDependencies  ++= Seq(
  // Last stable release
  "org.scalanlp" %% "breeze" % "0.13.2",

  // Native libraries are not included by default. add this if you want them (as of 0.7)
  // Native libraries greatly improve performance, but increase jar sizes.
  // It also packages various blas implementations, which have licenses that may or may not
  // be compatible with the Apache License. No GPL code, as best I know.
  "org.scalanlp" %% "breeze-natives" % "0.13.2",

  // The visualization library is distributed separately as well.
  // It depends on LGPL code
  "org.scalanlp" %% "breeze-viz" % "0.13.2"
)
```

这里其官方 Github 的文档还有这样一句：

```sbt
resolvers += "Sonatype Releases" at "https://oss.sonatype.org/content/repositories/releases/"
```

但是我加上以后就会报错，去掉以后就 build 好了，很奇怪。

## 快速开始
首先引入下面内容：

```scala
import breeze.linalg._
```

新建一个向量：

```scala
// Dense vector will allocate memory for zeros, where sparse won't.
val x = DenseVector.zeros[Double](5)
val y = SparseVector.zeros[Double](5)

println(x) // DenseVector(0.0, 0.0, 0.0, 0.0, 0.0)
println(y) // SparseVector(5)()
```

**与 Scala 不同，breeze 创建的所有向量都是列向量**。

可以通过下标获取或者更改向量中的元素，而且支持负数下标。

```scala
// Access and update data elements by their index
// and negative indices are supported, x(i) == x(x.length + i)
println(x(0)) // 0.0
x(4) = 4
println(x(-1)) // 4.0
```

Breeze 也支持切片，并且**范围切片比任意切片快得多**。切片赋值时使用的时 `:=`，可以给切片赋一个相同的值，也可以给其赋一个相同长度的向量。

```scala
x(3 to 4) := .5
x(0 to 1) := DenseVector(.1, .2)
println(x) // DenseVector(0.1, 0.2, 0.0, 0.5, 0.5)
```

接下来我们再新建一个全为 0 的 5x5 矩阵：

```scala
val m = DenseMatrix.zeros[Int](5, 5)
println(m)
```

结果：

```scala
0  0  0  0  0  
0  0  0  0  0  
0  0  0  0  0  
0  0  0  0  0  
0  0  0  0  0
```

然后我们可以访问其行或者列，列就是 `DenseVectors` 的类型，行是 `DenseVectors` 的转置。

```scala
// The columns of m can be accessed as DenseVectors, and the rows as DenseMatrices.
val size = (m.rows, m.cols)
println(size) // (5,5)

val column = m(::, 1)
println(column) // DenseVector(0, 0, 0, 0, 0)

// Transpose to match row shape
m(4, ::) := DenseVector(1, 2, 3, 4, 5).t
println(m)
```

结果：

```scala
0  0  0  0  0  
0  0  0  0  0  
0  0  0  0  0  
0  0  0  0  0  
1  2  3  4  5
```

同样可以切割出一个子矩阵：

```scala
m(0 to 1, 0 to 1) := DenseMatrix((3, 1), (-1, -2))
println(m)
```

结果：

```scala
3   1   0  0  0  
-1  -2  0  0  0  
0   0   0  0  0  
0   0   0  0  0  
1   2   3  4  5
```
