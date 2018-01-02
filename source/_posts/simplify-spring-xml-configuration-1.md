---
layout:     post
title:      "简化 Spring XML 配置（一）"
subtitle:   "自动装配 bean 属性"
date:       2016-01-07
author:     "Ink"
catalog:    true
header-img: "http://ox2ru2icv.bkt.clouddn.com/image/post/post-bg-spring.jpg"
tags:
    - Spring
---

> 对于小的程序来说，在XML中配置的bean数目很少。但是如果对于一个大一点的应用，需要在XML中配置很多的bean。那么有没有简化XML配置的方法呢，Spring为我们提供了这个机制，现在就讲几种简化XML配置的方法。

装配一个bean的属性一般是用`<property>`元素，这里给出一种更简单的不需要配置`<property>`元素的方法——自动装配。
## 四种自动装配类型
#### 通过名字自动装配-byName
byName类型的装配方式是bean指在XML文件中搜索，当找到某个bean的名字（id）与其自身属性名字一样时，就自动装配。在XML配置的方式是在`<bean>`元素内加上autowire="byName"属性。

**no byName**

```xml
	<bean id="songJavaer2" class="com.springinaction.springidol.SongJavaer2">
		<property name="blogNum" value="30" />
		<property name="songJay" ref="songJay" />
	</bean>
	<bean id="songJay" class="com.springinaction.springidol.SongJay" />
```

**byName**

```xml
	<bean id="songJavaer2"
		class="com.springinaction.springidol.SongJavaer2"
		autowire="byName">
		<property name="blogNum" value="30" />
	</bean>
	<bean id="songJay" class="com.springinaction.springidol.SongJay" />
```

#### 通过类自动装配-byType
通过类自动装配是指，bean搜索XML文件，只要找到与其属性的类相同的其他bean，就自动装配。在XML配置的方式是在`<bean>`元素内加上autowire="byType"属性。

```xml
	<bean id="songJavaer2"
		class="com.springinaction.springidol.SongJavaer2"
		autowire="byType">
		<property name="blogNum" value="30" />
	</bean>
```

但是现在有一个问题，如果在搜索的过程中，不止找到一个与其属性的类相同的bean，这种情况下程序就会报错。为了解决这个问题，可以使用两个属性：primary和autowire-candidate。

primary用来指明一个bean是否是被装配的第一选择，比如当设置primary="true"以后，就是告诉那个需要装配的bean，先选我先选我，我是最好的。然后，有一个很有趣的事情，Spring中将primary的默认值设为true，这就意味这所有供选择的bean都是“最好的”，这样还是无法选择，因此必须将除了一个之外的其他所有bean的primary设置为false，如下：

```xml
	<bean id="songJay"
		  class="com.springinaction.springidol.SongJay"
		  primary="false"/>
```

autowire-candidate用来指明bean是否作为“候选人”，当设置为false时就不会选择装配这个bean了，如下：

```xml
	<bean id="songJay"
		  class="com.springinaction.springidol.SongJay"
		  autowire-candidate="false"/>
```

#### 自动装配构造器-constructor
是指自动搜索与构造器中的参数的类相同的bean。在XML中的配置方式是在`<bean>`元素内加上autowire="constructor "属性，如下：

```xml
	<bean id="songJavaer2"
		class="com.springinaction.springidol.SongJavaer2"
		autowire="constructor ">
	</bean>
```

#### 自适应的自动装配方式-autodetect
首先尝试使用constructor的自动装配方式，如果失败的话就再次尝试使用byType的自动装配方式，如下：

```xml
	<bean id="songJavaer2"
		class="com.springinaction.springidol.SongJavaer2"
		autowire="autodetect ">
	</bean>
```

---

## 设置默认装配方式
如果你想要在你的XML文件中使用同一种自动装配方式来装配bean，那么可以使用默认装配的方式，设置方法是在XML中的`<beans>`设置default-autowire属性，如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans-3.0.xsd"
default-autowire="byType">
</beans>
```

说明两点，一是这种方式仅限当前XML文件中，而不是整个应用环境上下文；二是即使配置了默认自动装配方式，但具体的bean你还是可以专门配置来覆盖这个默认配置。

---

## 将自动装配和手动装配混合起来使用
即使使用了自动装配以后还是可以混合手动装配来使用，这样可以解决一些问题，例如使用byType时候如果找到很多个相同类的bean，那么可以使用手动加入`<property>`元素的方式来指定具体的某个bean，还可以给自动装载的bean初始化为null，如下：

```xml
	<bean id="kenny"
		class="com.springinaction.springidol.Instrumentalist"
	autowire="byType">
		<property name="song" value="Jingle Bells" />
		<property name="instrument"><null/></property>
	</bean>
```

但是注意对于构造器注入来说，只能使用自动装配或者手动装配，不能混合使用。
