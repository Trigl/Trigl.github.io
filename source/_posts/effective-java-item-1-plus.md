---
layout:     post
title:      "Effective Java 读书笔记：Item 1（续）"
subtitle:   "服务提供者框架"
date:       2019-08-26
author:     "Ink Bai"
header-img: "/img/post/effective-java-1-plus.jpg"
header-style: "text"
tags:
    - Effective Java
---

> 上一篇 [Effective Java 读书笔记：Item 1](https://baixin.ink/2019/08/25/effective-java-item-1/) 中留了一个小坑，Effective Java 这本书里在谈到静态工厂方法的第五个优点时，是这样写的：
A fifth advantage of static factories is that the class of the returned object need not exist when the class containing the method is written. Such flexible static factory methods form the basis of service provider frameworks, like the Java Database Connectivity API (JDBC).

这里面说的 `service provider frameworks` 即服务提供者框架是个什么东东呢？

服务提供者框架是指：系统会提供一个服务，这个服务的实现方式可以有很多种，具体的实现方式就由服务提供者来实现，客户端可以自由使用任意一个服务提供者提供的服务，在客户端看来调用不同的服务实现的方式都是一样的，即客户端和具体的服务实现是解藕的。

从字面上来看，服务提供这框架主要有系统、服务和服务提供者，对应到程序中通常有四个组件：

- Service Interface：服务接口，将服务通过抽象统一声明，供客户端调用、由各个服务提供者具体实现。
- Provider Registration API：服务提供者注册 API，用于系统注册某个具体的服务提供者，使得客户端可以使用它实现的服务。
- Service Access API：服务访问 API，用户客户端获取相应的服务。
- Service Provider Interface（可选）：服务提供者接口，这些服务提供者负责服务的具体实现。

Java 里面的 JDBC 就是服务提供者框架，这里我们通过 JDBC 创建 mysql 的数据库连接的例子来讲一下服务提供者框架：

```java
// 首先要导入数据库驱动jar包、直接复制到文件那里，然后加入到路径
// 1. 注册驱动，java反射机制
Class.forName("com.mysql.jdbc.Driver");
// 2. 创建一个连接对象
Connection conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/mydb", "root", "password");
// 3. 创建一个sql语句的发送命令对象
Statement stmt = conn.createStatement();
// 4. 执行sql,拿到查询的结果集对象
ResultSet rs = stmt.executeQuery("select * from stu");
// 5. 输出结果集的数据
while (rs.next()) {
    System.out.println(rs.getInt("id") + ":" + rs.getString("name"));
}
// 6. 关闭连接，命令对象以及结果集。
rs.close();
stmt.close();
conn.close();
```

这里 `Connection` 接口就是一个服务接口，定义了很多操作供客户端调用。客户端通过 `DriverManager.getConnection` 获取 JDBC 连接，它就是服务访问 API。但是 JDBC 本身不对该服务进行实现，而是有服务提供者接口去具体实现，`Driver` 就是服务提供者接口，如 mysql 的 `com.mysql.jdbc.Driver` 就是 `Driver` 的具体实现。`DriverManager.registerDriver` 是服务提供者注册 API，但是看上面 JDBC 注册驱动的代码是这样的：

```java
Class.forName("com.mysql.jdbc.Driver");
```

说好的用 `DriverManager.registerDriver` 进行注册呢？不要急，来看一下 `com.mysql.jdbc.Driver` 的源码：

```java
public class Driver extends NonRegisteringDriver implements java.sql.Driver {  
    // Register ourselves with the DriverManager   
    static {  
        try {  
            java.sql.DriverManager.registerDriver(new Driver());  
        } catch (SQLException E) {  
            throw new RuntimeException("Can't register driver!");  
        }  
    }  
}
```

从上边可以看到，它是用静态代码块实现的。根据类加载机制，当执行 `Class.forName(driverClass)` 获取其Class对象时， `com.mysql.jdbc.Driver` 就会被 JVM 加载，连接，并进行初始化，初始化就会执行静态代码块，也就会执行下边这句代码：

```java
java.sql.DriverManager.registerDriver(new Driver());
```

所以实际上还是使用的 `DriverManager.registerDriver` 进行的服务注册。

综上所诉，服务提供者框架是这样的，首先会有一个官方标准制定者，制定一个具体的服务标准，然后定义一个类似于字典的服务管理器，给外面提供一个注册的接口，你实现了服务以后自己注册到字典里面去。然后会有具体的服务实现厂商，他们只要做两件事就可以了：1、根据官方标准实现服务。2、注册到官方服务管理器中。对于我们这些普通用户而言，我只要知道怎么用就行了，至于你们内部是怎么实现的我并不 care。

另外还有一点，Java 从 1.6 开始，还贴心地提供了一个通用的服务提供者框架 `java.util.ServiceLoader` 方便我们编程，具体的实例可以参考这篇文章：[Java Service Provider Interface](https://www.baeldung.com/java-spi)
