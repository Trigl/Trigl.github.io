---
layout:     post
title:      "SBT 那些常用的功能"
date:       2018-08-20
author:     "Ink Bai"
catalog:    true
header-img: "/img/post/sbt.jpg"
tags:
    - Scala
---

> 使用 Scala 也很久了，SBT 是最方便的构建工具，但是总是会遇到一些 SBT 引起的奇奇怪怪的问题，今天就好好研究一下 SBT 的常见用法，争取通过本文跨过大部分的坑。

## SBT 常见命令
**clean**：移除 `target` 文件夹下生成的所有文件。

**compile**：编译 `src/main/scala`，`src/main/java` 和项目根目录下的文件。

**run**：编译代码然后运行项目中的 `main` 方法，如果有多个，SBT 会要求你选一个。

**package**：将 `src/main/scala`，`src/main/java`，`src/main/resources` 下的文件打包成一个 jar 包。

**test**：编译及运行所有的测试用例。

**doc**：对你的 Scala 源码生成相应的 API 文档。

**reload**：重新加载所有与 sbt build 相关的文件，如 `build.sbt`，`project/*`。

**publish**：将你的项目发布到远程仓库。

**publishLocal**：将项目发布到本地 Ivy 仓库。

## 如何查看执行的命令的关键输出
使用 `last` 命令可以查看最后执行的命令的关键输出，如 `warn` 或者 `error` 信息。例如如果使用 `compile` 进行了编译，输出了一堆日志最后报错了，想要看到哪些部分有问题，就可以使用 `last compile` 命令打印出来。

## 使用 SBT 和 ScalaTest 测试
首先在 `build.sbt` 中加入 scalatest 的包：

```scala
"org.scalatest" %% "scalatest" % "3.0.5" % Test
```

然后在 `src/main/scala` 下创建一个 `Hello.scala` 的源文件：

```scala
package ink.baixin.scalalearning.sbt

object Hello extends App {
  val p = Person("Ink Bai")
  println("Hello from " + p.name)
}

case class Person(var name: String)
```

然后在 `src/test/scala` 下写相应的测试用例：

```scala
package ink.baixin.scalalearning.sbt

import org.scalatest.FunSuite

class HelloTests extends FunSuite {
  test("the name is set correctly in constructor") {
    val p = Person("Ink Bai")
    assert(p.name == "Ink Bai")
  }

  test("a Person's name can be changed") {
    val p = Person("Will")
    p.name = "William"
    assert(p.name == "William")
  }
}
```

现在就可以使用 sbt 的 `test` 命令来执行该测试用例了，如下：

```scala
$ sbt test

...
[info] HelloTests:
[info] - the name is set correctly in constructor
[info] - a Person's name can be changed
[info] Run completed in 581 milliseconds.
[info] Total number of tests run: 2
[info] Suites: completed 1, aborted 0
[info] Tests: succeeded 2, failed 0, canceled 0, ignored 0, pending 0
[info] All tests passed.
[success] Total time: 6 s, completed Aug 22, 2018 11:55:13 AM
```

## 如何管理依赖包？
在 `build.sbt` 中配置行之间必须有空白行来分割，下面是包含一个依赖包的简单但是全面的文件：

```scala
name := "scala-learning"

version := "0.1"

scalaVersion := "2.12.4"

libraryDependencies ++= "org.scalatest" %% "scalatest" % "3.0.5" % Test
```

如果想要添加多个依赖包，就可以定义一个 `Seq`：

```scala
libraryDependencies ++= Seq(
  "io.netty" % "netty-all" % "4.1.25.Final",
  "org.apache.calcite" % "calcite-core" % "1.16.0",
  "org.scalatest" %% "scalatest" % "3.0.5" % Test
)
```

从上面的例子可以推断出，`build.sbt` 的入口其实是简单的键值对，其中 `name`，`version`，`scalaVersion` 等就是最常见的 SBT 的键，SBT 通过创建一个很大的 Map 来构建项目。

这种方式就是管理依赖的方式（直接把 jar 包放到项目的 `lib` 目录下面的是非管理依赖的方式），可以自动下载管理某个 jar 包需要的其他依赖包。SBT 底层使用 Apache Ivy 来管理依赖，Ivy 也可以用于 Ant 和 Maven，因此你可以通过 SBT 在 Scala 项目中方便地使用 Java 的那些包。

一般有两种添加依赖包的方式，第一种需要指定 `groupID`，`artifactID` 和 `revision`：

```scala
libraryDependencies += groupID % artifactID % revision
```

实例如：

```scala
libraryDependencies += "org.scalatest" % "scalatest_2.12" % "3.0.5"
```

第二种多了一个可选的参数 `configuration`：

```scala
libraryDependencies += groupID % artifactID % revision % configuration
```

还是上面的例子：

```scala
libraryDependencies += "org.scalatest" % "scalatest_2.12" % "3.0.5" % Test
```

可以看到多了一个 `Test`，这个说明只是用于 test 的时候，这个配置可以理解为类似于 Maven 中的 scope。如下就是相应的 Maven 配置：

```scala
<dependency>
  <groupId>org.scalatest</groupId>
  <artifactId>scalatest_2.12</artifactId>
  <version>3.0.5</version>
  <scope>test</scope>
</dependency>
```

可以看到上面的 `scalatest_2.12` 有个 `2.12`，这个其实是 Scala 的版本，只有基于 Scala 的 Jar 包才会有这个依赖的 Scala 版本号的信息。而我们在 `build.sbt` 中是通过 `scalaVersion` 设置了 Scala 版本的，那么能不能把这个 `2.12` 省去或者使用 `scalaVersion` 这个参数设置呢，答案是肯定的，通过下面这种形式即可：

```scala
libraryDependencies += groupID %% artifactID % revision % configuration
```

细心的读者应该可以看出，第一个 `%` 变成了 `%%`，在 `groupID` 和 `artifactID` 之间加 `%%` 之后，`artifactID` 就不需要指定 Scala 版本了，SBT 会自动将其设置的 Scala 版本添加到这个值的后面，如下：

```scala
scalaVersion := "2.12.4"

libraryDependencies += "org.scalatest" %% "scalatest" % "3.0.5" % Test
```

就等价于：

```scala
libraryDependencies += "org.scalatest" % "scalatest_2.12" % "3.0.5" % Test
```

## 如何给一个项目创建子项目
首先创建两个子项目，目录结构如下：

```scala
├── subproject1
│   ├── build.sbt
│   └── src
│       └── main
│           └── scala
│               └── Foo.scala
├── subproject2
│   ├── build.sbt
│   └── src
│       └── main
│           └── scala
│               └── Bar.scala
```

其中子项目的 `build.sbt` 如下：

```scala
// name := "scala-learning subproject2"
name := "scala-learning subproject1"

version := "0.1"

scalaVersion := "2.12.4"
```

`Foo.scala` 如下：

```scala
package ink.baixin.scalalearning.subproject1

case class Foo (name: String)
```

`Bar.scala` 如下：

```scala
package ink.baixin.scalalearning.subproject2

import ink.baixin.scalalearning.subproject1._

case class Bar(name: String)

object HelloFoo extends App {
  println(Foo("Hello Foo"))
}
```

然后再在根目录下的 `build.sbt` 中增加如下内容：

```scala
lazy val subproject1 = (project in file("subproject1"))

lazy val subproject2 = (project in file("subproject2")).dependsOn(subproject1)

// aggregate: running a task on the aggregate project will also run it on the aggregated projects.
// dependsOn: a project depends on code in another project.
// without dependsOn, you'll get a compiler error: "object bar is not a member of package
// com.alvinalexander".
lazy val root = (project in file("."))
  .aggregate(subproject1, subproject2)
  .dependsOn(subproject1, subproject2)
```

这里可以看到首先定义了 `subproject1` 和 `subproject2` 分别代表两个子项目的 SBT，然后定义了 `root` 作为父目录的 SBT。其中 `dependsOn` 是指依赖关系，可以直接使用其依赖项目的代码；而 `aggregate` 是指如果连带关系，如果父项目执行某个任务如 `compile`，那么两个子项目也会相应执行该任务。

我们在父项目中写一个对象来测试是否能够实现代码依赖：

```scala
package ink.baixin.scalalearning.sbt

import ink.baixin.scalalearning.subproject2._
import ink.baixin.scalalearning.subproject1._

object MultipleProjectsExample extends App {
  println(Foo("I'm a Foo"))
  println(Bar("I'm a Bar"))
}
```

结果：

```scala
Foo(I'm a Foo)
Bar(I'm a Bar)
```

上面就是在 Scala 中创建多模块项目的例子，实际开发中可能会遇到一些大型项目内部业务比较复杂，模块比较多，那么我们就可以考虑把他们分成几个小部分，分出公共的部分作为一个依赖项目，这样可以使我们的代码更加清晰有条理。

## 如何指定一个 Main Class
可以通过添加如下内容来设置 main class：

```scala
// set the main class for packaging the main jar
mainClass in (Compile, packageBin) := Some("ink.baixin.scalalearning.sbt.Hello")

// set the main class for the main 'sbt run' task
mainClass in (Compile, run) := Some("ink.baixin.scalalearning.sbt.Hello")
```

## 如何使用 github 中的代码
现在我们想直接使用一个 github 开源库的代码，可以如下配置：

```scala
lazy val githubProject = RootProject(uri("https://github.com/Trigl/akka-learning.git"))

lazy val root = (project in file(".")).dependsOn(githubProject)
```

这样设置以后就可以在代码中直接引用该开源库的代码了。

## 如何在 SBT 中设置 Ivy 仓库

可以在 `build.sbt` 中使用 `resolvers` 来添加一个 Ivy 仓库：

```scala
resolvers += "repository name" at "location"
```

示例如下：

```scala
resolvers += "Java.net Maven2 Repository" at "http://download.java.net/maven/2/"
```

当然也可以用 `Seq` 添加多个仓库：

```scala
resolvers ++= Seq(
  "Typesafe" at "http://repo.typesafe.com/typesafe/releases/",
  "Java.net Maven2 Repository" at "http://download.java.net/maven/2/"
  )
```

## 如何设置 SBT 的日志等级
这个简单，添加如下内容即可：

```scala
logLevel := Level.Info
```

## 如何部署一个完整可执行的 Jar 包
这个问题看起来很 easy 嘛，直接使用 `sbt package` 不就得了，实际上是不行的，因为 `package` 命令只会把本项目 `src/main/scala`，`src/main/java`，`src/main/resources` 下的文件打成一个 Jar 包，而有两类重要且必须的内容却不会打包，这两类是：

- 项目的依赖包，即在 `build.sbt` 中定义的那些依赖包。
- 用来在分布式环境中执行时所需要的 Scala 的 Jar 包，因为一般分布式环境中是通过 Java 来执行的，这个时候就需要指定 Scala 的包才可以执行 Scala 程序。

那么针对这个问题如何解决呢，我们推荐的方法就是使用开源工具 [sbt-assembly](https://github.com/sbt/sbt-assembly) 来打包，这个项目会把需要的所有依赖都包含进来，部署一个在分布式环境可执行的 Jar 包。

那么如何使用呢，首先在 `project` 目录下面添加文件 `assembly.sbt`：

```scala
addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "0.14.7")
```

然后执行 `sbt assembly` 就可以打包了，就是这么简单，当然还可以在 `build.sbt` 中添加一些更加详细的配置。

## 如何发布你的代码
发布代码可以使用命令 `publish` 或者 `publishLocal`，`publish` 是将项目发布到一个远程仓库，所以需要设置远程仓库的地址，现在我设置一个本机路径作为仓库地址：

```scala
// publish project to a remote repository
publishTo in ThisBuild := Some(Resolver.file("file", new File("/Users/will/tmp")))
```

然后执行 `sbt publish` 结果如下：

```scala
...
[info] 	published scala-learning_2.12 to /Users/will/tmp/scala-learning/scala-learning_2.12/0.1.part/scala-learning_2.12-0.1.pom
[info] 	published scala-learning_2.12 to /Users/will/tmp/scala-learning/scala-learning_2.12/0.1.part/scala-learning_2.12-0.1.jar
[info] 	published scala-learning_2.12 to /Users/will/tmp/scala-learning/scala-learning_2.12/0.1.part/scala-learning_2.12-0.1-sources.jar
[info] 	published scala-learning_2.12 to /Users/will/tmp/scala-learning/scala-learning_2.12/0.1.part/scala-learning_2.12-0.1-javadoc.jar
[info] 	publish commited: moved /Users/will/tmp/scala-learning/scala-learning_2.12/0.1.part
[info] 		to /Users/will/tmp/scala-learning/scala-learning_2.12/0.1
[success] Total time: 1 s, completed Aug 22, 2018 4:13:46 PM
```

另外一个发布命令是 `publishLocal`，会将项目发布到本地的 Ivy 仓库，在本机的其他项目中可以直接调用这个包，只要你在其他项目的 `build.sbt` 中写入这个包的依赖即可，如下；

```scala
libraryDependencies += "scala-learning" %% "scala-learning" % "0.1"
```

## Refer
[Scala Cookbook](https://book.douban.com/subject/20876182/)
[build.sbt 示例](https://www.scala-sbt.org/1.0/docs/Basic-Def-Examples.html)
[项目源码](https://github.com/Trigl/scala-learning)
