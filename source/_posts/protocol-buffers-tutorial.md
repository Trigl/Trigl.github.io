---
layout:     post
title:      "Protocol Buffers 了解一下？"
date:       2018-06-20
author:     "Ink Bai"
catalog:    true
header-img: "http://ox2ru2icv.bkt.clouddn.com/image/post/protocol-buffers.jpg"
tags:
    - Big Data
---

> 本文是学习 Protocol Buffers 时做的笔记，内容来自[官方文档](https://developers.google.com/protocol-buffers/)，把其中的精华部分提炼出来做了一个总结。

## 什么是 protocol buffers
Protocol buffers 是一种跨语言跨平台可扩展的序列化结构化数据的方式，常用于通信协议、数据存储等等。首先会定义数据应当如何构造，然后使用特殊生成的源代码把结构化的数据写入到各个数据流或者读取出来。甚至可以更新数据结构而不会破坏已发布的数据格式。

## 如何工作？
我们可以在 `.proto` 文件中定义要序列化的数据结构和消息类型，每一个 protocol buffer 都是包含信息的记录，由一系列键值对组成。下面是定义 `Person` 信息的 `.proto` 文件的例子：

```
message Person {
  required string name = 1;
  required int32 id = 2;
  optional string email = 3;

  enum PhoneType {
    MOBILE = 0;
    HOME = 1;
    WORK = 2;
  }

  message PhoneNumber {
    required string number = 1;
    optional PhoneType type = 2 [default = HOME];
  }

  repeated PhoneNumber phone = 4;
}
```

可以看到消息格式很简单，每一个消息类型都有一个或者多个单独的数字化的字段。字段有自己的名字和类型，类型包括数字（整数或浮点数）、boolean、string、byte 甚至其他 protocol buffer 消息类型，这样可以定义分层的数据。而且字段可以加 `optional`，`required` 和 `repeated` 修饰符，关于 `.proto` 文件格式的具体内容我们后文再讲。

一旦定义好消息以后，就可以运行 protocol buffer complier，生成你所使用的语言的 `.proto` 文件的数据实体类。这些类可以提供消息字段的入口（例如 `name()` 和 `set_name()`）以及将消息记录序列化为字节或者从字节解析成消息记录的方法。例如，如果你使用的语言是 C++，对上面的例子执行 protocol buffer complier 将会生成一个称为 `Person` 的类，你可以在你的应用中使用这个类来构成、序列化或者恢复 protocol buffer 的消息记录。你可以写下面这样的代码来序列化消息：

```C++
Person person;
person.set_name("John Doe");
person.set_id(1234);
person.set_email("jdoe@example.com");
fstream output("myfile", ios::out | ios::binary);
person.SerializeToOstream(&output);
```

然后可以再解析出消息：

```C++
fstream input("myfile", ios::in | ios::binary);
Person person;
person.ParseFromIstream(&input);
cout << "Name: " << person.name() << endl;
cout << "E-mail: " << person.email() << endl;
```

可以给消息格式增加新的字段而不会破坏向后兼容性，老的代码在解析的时候会忽略掉新的字段。

## 为什么不使用 XML？
序列化结构化数据时，protocol buffers 比 XML 有更多的优势：

- 更简单
- 小 3-10 倍
- 快 20-100 倍
- 更加清晰
- 会生成数据实体类，更易于编程

例如在 XML 中如果想模块化一个具有 `name` 和 `email` 的 `Person`，你需要：

```xml
<person>
  <name>John Doe</name>
  <email>jdoe@example.com</email>
</person>
```

而在 protocol buffer 中是这样的：

```
# Textual representation of a protocol buffer.
# This is *not* the binary format used on the wire.
person {
  name: "John Doe"
  email: "jdoe@example.com"
}
```

当这条消息被编码成 protocol buffer 的二进制格式后（上面代码是文本格式展现，仅仅是为了方便人类阅读、debug 和编辑），大小只有 28 bytes，解析需要 100-200 纳秒。而 XML 版本的大小至少有 69 bytes，需要 5000-10000 纳秒来解析。

而且操作一个 protocol buffer 也更容易：

```
cout << "Name: " << person.name() << endl;
cout << "E-mail: " << person.email() << endl;
```

同样地操作放在 XML 上：

```
cout << "Name: "
     << person.getElementsByTagName("name")->item(0)->innerText()
     << endl;
cout << "E-mail: "
     << person.getElementsByTagName("email")->item(0)->innerText()
     << endl;
```

当然，并不是所有的情况下 protocol buffer 都比 XML 好，例如 protocol buffer 并不擅长格式化基于文本的标记语言（如 HTML）。此外，XML 是人类可读和可编辑的，自身可以显示所包含的信息，而 protocol buffer 做不到这一点。

## Scala 中使用 protocol buffer
Protocol buffer 目前还不支持 Scala，但是有开源的 [ScalaPB](https://scalapb.github.io) 可以使用。我们使用一个简单的 “通讯录” 的例子来讲解，这个应用可以从文件中读取和写入联系人信息，通讯录中的每个人都有名字，ID，电子邮箱和电话号码。
#### 为什么使用 Protocol Buffers？
如何序列化和恢复数据呢？有以下几种方法：

- 使用 Java 序列化。Java 默认的序列化方式，但是存在一个众所周知的问题（*Effective Java* 中提到），并且不是跨语言的。
- 你可以发明一些特殊的方式将数据编码成单一字符，例如将 “12:3:-23:67” 编码成数字 4。这种方式简单灵活，但是代码可能是一次性的，无法复用，仅适用于编码非常简单的数据。
- 将数据序列化成 XML。优点是可读性好并且兼容大部分语言，缺点也很明显，首先是数据量很大，其次是 XML DOM 树形结构用起来没有类用起来方便。

综上所诉，protocol buffers，你值得拥有。

#### 在 SBT 中安装 ScalaPB 插件
首先创建包含下面内容的文件 `project/scalapb.sbt`：

```
addSbtPlugin("com.thesamet" % "sbt-protoc" % "0.99.18")

libraryDependencies += "com.thesamet.scalapb" %% "compilerplugin" % "0.7.4"
```

然后将下面内容加到 `build.sbt`：

```
PB.targets in Compile := Seq(
  scalapb.gen() -> (sourceManaged in Compile).value
)

// (optional) If you need scalapb/scalapb.proto or anything from
// google/protobuf/*.proto
libraryDependencies += "com.thesamet.scalapb" %% "scalapb-runtime" % scalapb.compiler.Version.scalapbVersion % "protobuf"
```

ScalaPB 在 `src/main/protobuf` 文件夹下找 `.proto` 文件，当然这个可以自定义配置。在 sbt 下运行 `compile` 将会从 protos 生成 Scala 源文件并且编译。

#### 定义 Protocol 格式
创建 “通讯录” 应用的第一步就是写 `.proto` 文件。定义 `.proto` 文件很简单，为每一个你想序列化的数据结构添加一个 `message`，然后在这个 `message` 下定义字段，每个字段包含名字和类型，如下面就是 “通讯录” 的 `models.proto`：

```
syntax = "proto3";

package ink.baixin.scalalearning;

message Person {
  string name = 1;
  int32 id = 2;
  string email = 3;

  enum PhoneType {
    MOBILE = 0;
    HOME = 1;
    WORK = 2;
  }

  message PhoneNumber {
    string number = 1;
    PhoneType type = 2;
  }

  repeated PhoneNumber phones = 4;
}

message AddressBook {
  repeated Person people = 1;
}
```

第一行显示使用的 protocol buffer 的版本，有 proto2 和 proto3，这两个版本的区别还是很大的，这里我们使用 proto3。proto3 字段默认是 `optional`，并且不允许添加 `optional`，所有的字段都不能设置为 `required`，并且字段不能有默认值。

然后就是指定 `package`，自动生成的类所在的包就是 `package.base_name`，其中 `base_name` 就是 `.proto` 文件的名称，所以这个例子中自动生成的类所在的包名就是 `ink.baixin.scalalearning.models`

接下来显示的就是 message 的定义。一个 message 就是包含一系列类型字段的聚合。许多标准简单数据类型可以作为字段类型，包括 bool、int32、float、double 和 string。你也可以用其他 message 类型作为一个 message 的字段类型，如上面 message `Person` 包含 `PhoneNumber` 类型的字段，而 message `AddressBook` 包含 `Person` 类型的字段。你也可以在一个 message 中定义另一个 message，如上面 message `PhoneNumber` 是定义在 message `Person` 内部的。你也可以使用 `enum` 类型如果希望某个字段的值是一个预定义值列表中的一个，如上面指定了 `PhoneNumber` 的 `type` 是 `MOBILE`、`HOME` 或者 `WORK` 中的一个。

上面的 `=1`，`=2` 是字段用于二进制编码的唯一标记，标记数字 0-15 仅占一个字节的长度，所以应当优先用于那些经常使用的字段或者 `repeated` 的字段，而超过 15 的标记数字可以用于不那么常用的可选字段。

#### 编译 protocol buffer
在 sbt 运行 `compile`，之后会在 `$PROJECT_PATH/target/scala-2.1x/src_managed/main` 下生成 protoc 编译后的类文件，我们可以使用这些类对 messege 进行相关操作。

#### 写入并读取 message
下面我们看一下如何使用 protocol buffer 将 message 写入一个文件，然后再从这个文件读取这些消息。

```scala
object ProtocExample extends App {
  val p1 = Person(name = "Ink", id = 0, email = "ink@example.com")
  val p2 = p1.update(
    _.name := "Baixin",
    _.email := "baixin@example.com"
  )

  val a = AddressBook(Seq(p1, p2))
  println(a)

  val f = new File("address.txt")
  f.createNewFile()
  val oF = new FileOutputStream(f, false)
  oF.write(a.toByteArray)
  oF.close

  val iF = new FileInputStream(f)
  val a1 = AddressBook.parseFrom(iF)
  println(a1)
}
```

## 总结
Protocol buffer 在谷歌内部广泛用于数据传输和存储，所以其通用性应该是毋庸置疑的。我们可以在网络传输、数据存储时使用 protocol buffer 来序列化数据，例如把数据写入 HDFS 中或者从 Kafka 读取数据时，都可以把数据使用 protocol buffer 序列化，可以有效减少数据量、降低带宽磁盘等压力。
