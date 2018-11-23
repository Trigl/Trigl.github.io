---
layout:     post
title:      "Summary of Spring"
subtitle:   "《Spring in Action》笔记，总体介绍一下Spring"
date:       2015-11-25
author:     "Ink Bai"
catalog:    true
header-img: "/img/post/post-bg-spring.jpg"
tags:
    - Spring
---

> 本章内容是从整体理论高度上介绍Spring，有些概念不是很明白，暂时可以不求甚解，留待日后慢慢消化。

## 为什么使用Spring？
Java语言诞生以后，开发者就开始使用它来创造并且丰富动态的web应用。

之后，Sun公司发布JavaBeans的规范。JavaBeans定义了用于Java的软件组件模型。这个规范定义了一系列确保简单的Java对象能够被复用并且组合成更多复杂应用的编程协议。

尽管JavaBeans的功能已经很方便了，但是企业级开发者还想要实现更多功能，例如，一个复杂的应用必定包含事务处理、安全性、分布式处理功能，但是这些在JavaBeans的官方规范中是没有的，所有之后Sun公司发布了Enterprise JavaBeans（EJB）规范，用于实现这些功能。然而EJB与原始的JavaBeans除了名字以外几乎没有什么相似之处。

EJB的确可以实现很多功能，企业级开发者也通过应用EJB实现了很多功能，然而EJB本身存在很多缺点，比如冗余而庞大的框架结构，框架中有很多东西都是没有实际用途但是却为了维护EJB框架本身而必须存在的东西，这样就导致了一个问题，框架中的代码比实现功能所需的核心代码还要多，失去了Java语言最原始的简洁朴素性，所以慢慢地，EJB开始没落。

如今，人们更倾向于使用基于**plain-old Java objects ( POJOs)**即原始的简洁的Java对象的编程模型，结合一些**EJB的功能**（但是去除EJB中复杂的东西），以及包括**AOP**（aspect-oriented programming）即面向切面编程和**DI**（dependency injection）依赖注入等一些编程技术，来实现企业级应用。而这些就是Spring框架所实现的。

---

## Spring的核心技术
Spring最大的优点就是简化Java开发。主要通过以下四种方式实现。
#### 基于POJOs的开发
前文提到，POJOs就是plain-old Java objects，即普通而古老的Java对象，Spring尽量遵从原始Java语法的简洁性，让我们在使用Spring的时候甚至没有意识到已经使用了Spring。

与之相对应的是EJB，EJB本身的生命周期机制就迫使你必须使用对你的程序来说没有用的接口，这样就完全是程序为框架服务而不是框架为程序服务了。这样就会产生很多没有用的冗长的垃圾代码。
#### 注入依赖
一个类的数据成员可能是另一个类，或者你的成员函数中有其他类作为参数，此时当要实例化当前类，必须首先实例化数据成员或者成员函数中的参数类，这些类就叫做当前类的依赖，简而言之，依赖就是实例化当前类时需要先实例化的类。

例如如下的一个类:


```java
package com.springinaction.knights;

public class DamselRescuingKnight implements Knight {
  private RescueDamselQuest quest;
  public DamselRescuingKnight() {
    quest = new RescueDamselQuest();
  }
  public void embarkOnQuest() throws QuestException {
    quest.embark();
  }
}
```

其中RescueDamselQuest 是数据成员，当前类的构造函数需要实例化RescueDamselQuest ，它就是当前类的一个依赖。

BTW，再来讲一下什么是内聚和耦合。内聚（cohesion）是一个模块内部各成分之间相关联程度的度量，耦合（Coupling）是模块之间依赖程度的度量。内聚和耦合是密切相关的，与其它模块存在强耦合的模块通常意味着弱内聚，而强内聚的模块通常意味着与其它模块之间存在弱耦合。模块设计追求强内聚，弱耦合。Spring以在XML文件中写bean的形式代替原始Java程序在实例化的时候直接实例化其依赖，这种方式就减小了耦合性，使程序的复用性提高。

#### AOP技术
AOP（aspect-oriented programming）面向切面的编程经常被定义为在一个系统中促进系统各部分和某些非核心功能点分离的技术。一个系统由很多部分组成，而每一个部分负责一个主要的功能，但是每一部分也会需要其他额外的系统功能例如安全性、日志功能和事物管理功能，这些功能是横向切割的功能，因为在一个系统的多个部分中它们都存在。

如图1.2，一个系统内部的各个成分都与右边的系统性功能有关联，同时它们又有自身独特的功能，但是这么多功能联合在一起显得杂乱无章。

![这里写图片描述](/img/content/20151128174039995.png)

AOP编程确保将系统性服务形成一个模块，然后分层次应用到系统的各个部分，这样对于系统各部分来说，这些系统性服务对它们是一层层的毯子，甚至是透明的。

![这里写图片描述](/img/content/20151128174139412.png)

#### 缩减冗余模板式代码
模板式代码就是指程序必须写的且经常用到的，但是又很模式化的代码，它们与具体的业务逻辑无关。最常见的就是Java连接jdbc的时候，每次代码中都有很多重复部分，只有在执行sql的部分有所不同，Spring将这些模板式的代码封装了起来，减少了我们在这些非业务事务上花费的无用的时间。

---

## Spring的工作区——容器
所有基于Spring的应用，其应用对象都是存活在Spring的容器中。一个Spring容器将会创建对象，将这些对象联系在一起，配置这些对象，并且管理这些对象的完整的生命周期（从创建到销毁）。容器是Spring框架的核心。所以，所有的Spring应用，创建容器是第一步，因为只有通过container我们才可以操控所有需要的对象。Spring的容器使用依赖注入管理组成应用的各个部分。不存在单一的Spring容器。Spring的容器有很多种，大体上可以分为两类：Bean factories和Application contexts。

- bean工厂：Bean factories（由org.springframework.beans.factory.BeanFactory接口定义）是最简单的容器，提供DI的基本支持。
- 环境上下文：Application contexts（由org.springframework.context.ApplicationContext接口定义）建立在bean工厂的概念之上，提供应用框架服务，比如从一个properties文件中解析文本信息的能力，或者给事件监听器发布应用事件的能力。

这里我们主要研究环境上下文。

#### 使用环境上下文加载容器
对于一个Spring应用，我们将所需要的bean定义在XML文件中，而所有的bean是在container中起作用的，所有第一步首先是创建container，我们经常遇到的创建container的方法有以下三种：

- classPathXmlApplicationContext：从位于classpath中的xml文件中加载环境上下文定义。举例如下：

  ```java
  ApplicationContext context = new
  ClassPathXmlApplicationContext("foo.xml");
  ```

  ClassPathXmlApplicationContext会默认从classpath的根目录寻找foo.xml

- FileSystemXmlApplicationContext：从文件系统中的XML文件中加载环境上下文定义。举例如下:

  ```java
  ApplicationContext context = new
  FileSystemXmlApplicationContext("c:/foo.xml");
  ```

  FileSystemXmlApplicationContext默认从我们的文件系统中寻找foo.xml

- XmlWebApplicationContext：从web应用包含的XML文件中加载环境上下文定义

#### bean的生命周期

![这里写图片描述](/img/content/20160104213743962.png)

图1.5展示了一个bean从创建到最后销毁的生命周期，具体步骤如下:

1. Spring实例化bean
2. Spring将具体的值和beans的引用注入到bean的properties中
3. 如果bean实现了BeanNameAware类，Spring会将bean的ID传递到setBeanName()方法中
4. 如果bean实现了BeanFactoryAware类，Spring会调用 setBeanFactory()方法，传递这个bean本身的bean工厂
5. 如果bean实现了ApplicationContextAware类，Spring将会调用setApplicationContext()方法，将引用传递到封装的应用上下文中
6. 如果任何bean实现了BeanPostProcessor类，Spring调用postProcessBeforeInitialization()方法，这是实例化之前的操作
7. 如果任何bean实现了InitializingBean类，spring将调用afterPropertiesSet()方法。同样的，如果bean中声明了一个初始化方法，那么将会调用一个特殊的初始化方法
8. 如果任何bean实现了BeanPostProcessor类，Spring调用postProcessAfterInitialization()方法，这是实例化之后的操作
9. 此时，bean已经准备好备应用调用了，并且在应用环境上下文被摧毁之前它会一直存在
10. 如果任何bean实现了DisposableBean接口，Spring将会调用destroy()方法。同时，如果bean中声明了一个销毁方法，那么Spring将调用以个特殊的销毁方法
