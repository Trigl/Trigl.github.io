---
layout:     post
title:      "Spring AOP（三）"
subtitle:   "通过 @AspectJ 注解切面"
date:       2016-03-17
author:     "Ink Bai"
catalog:    true
header-img: "/img/post/post-bg-spring.jpg"
tags:
    - Spring
---

> 在Spring AOP（二）中给出了在XML中配置切面的方法，本节学习通过@AspectJ来注解切面。

使用的例子仍然是Spring AOP（二）中的例子，前面讲的已经很详细了，具体原理不再说明，直接上代码
## 注解切面
**XML配置**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xmlns:aop="http://www.springframework.org/schema/aop"
xsi:schemaLocation="http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
http://www.springframework.org/schema/aop
http://www.springframework.org/schema/aop/spring-aop-3.0.xsd">
    <aop:aspectj-autoproxy />
    <bean id="drama" class="com.springinaction.aop.Drama" />
    <bean id="audience" class="com.springinaction.aop.Audience" />

</beans>
```

要点：

 - 添加`<aop:aspectj-autoproxy />`实现注解切面。
 - 切面和切点的Bean还是需要定义。

**注解切面**

```java
package com.springinaction.aop;

import org.aspectj.lang.annotation.AfterReturning;
import org.aspectj.lang.annotation.AfterThrowing;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;
@Aspect
public class Audience {

	@Pointcut("execution(* com.springinaction.aop.Performer.perform(..))")
	public void performance() {} // 定义切点

	@Before("performance()")
	public void takeSeats() {
		// 节目开始之前
		System.out.println("演出前——观众开始入座");
	}

	@Before("performance()")
	public void turnOffCellPhones() {
		// 节目开始之前
		System.out.println("演出前——观众关机或静音");
	}

	@AfterReturning("performance()")
	public void applaud() {
		// 节目成功结束之后
		System.out.println("演出很成功——观众鼓掌：啪啪啪");
	}

	@AfterThrowing("performance()")
	public void demandRefund() {
		// 节目表演失败之后
		System.out.println("节目演出很失败——切！一点都不好看，我们要求退钱！");
	}
}
```

要点：

 - 用@Aspect标注类
 - 用@Pointcut标注切点
 - 用@Before、@AfterReturning、@AfterThrowing标注前置、后置通知

**测试类**

```java
package com.springinaction.aop;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class AudienceTest {
	public static void main(String[] args) {
		ApplicationContext ctx = new ClassPathXmlApplicationContext(
		"com/springinaction/aop/aop1.xml");
		Performer drama = (Performer)ctx.getBean("drama");
		drama.perform();
	}
}
```

**输出结果**

```
演出前——观众开始入座
演出前——观众关机或静音
话剧正在进行中——（重复一千次）
演出很成功——观众鼓掌：啪啪啪
```

---

## 注解环绕通知
**切面类的环绕通知方法**

```java
	@Around("performance()")
	public void watchPerformance(ProceedingJoinPoint joinpoint) {
		try {
			System.out.println("演出前——观众开始入座");
			System.out.println("演出前——观众关机或静音");
			long start = System.currentTimeMillis();

			joinpoint.proceed(); // 执行被通知的方法

			long end = System.currentTimeMillis();

			System.out.println("演出很成功——观众鼓掌：啪啪啪");
			System.out.println("演出持续了 " + (end - start) + " milliseconds");
		} catch (Throwable e) {
			System.out.println("节目演出很失败——切！一点都不好看，我们要求退钱！");
		}
	}
```

**测试类**

```java
package com.springinaction.aop;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class AudienceTest {
	public static void main(String[] args) {
		ApplicationContext ctx = new ClassPathXmlApplicationContext(
		"com/springinaction/aop/aop1.xml");
		Performer drama = (Performer)ctx.getBean("drama");
		drama.perform();
	}
}
```

**输出结果**

```
演出前——观众开始入座
演出前——观众关机或静音
话剧正在进行中——（重复一千次）
演出很成功——观众鼓掌：啪啪啪
演出持续了 37 milliseconds
```

---

## 传递参数给注解的通知
**XML配置**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xmlns:aop="http://www.springframework.org/schema/aop"
xsi:schemaLocation="http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
http://www.springframework.org/schema/aop
http://www.springframework.org/schema/aop/spring-aop-3.0.xsd">
    <aop:aspectj-autoproxy />
    <bean id="magician" class="com.springinaction.aop.Magician" />
    <bean id="volunteer" class="com.springinaction.aop.Volunteer" />

</beans>
```

**切面**

```java
package com.springinaction.aop;

import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;

@Aspect
public class Magician implements MindReader {
	private String thoughts;

	@Pointcut("execution(* com.springinaction.aop.Thinker.thinkOfSomething(String)) "
			+ "&& args(thoughts)")
	public void thinking(String thoughts) {}

	@Before("thinking(thoughts)")
	public void interceptThoughts(String thoughts) {
		System.out.println("让我猜猜你在想什么？");
		this.thoughts = thoughts;
		System.out.println(this.getThoughts());
	}

	public String getThoughts() {
		System.out.print("你在想：");
		return thoughts;
	}

}
```

**测试类**

```java
package com.springinaction.aop;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class MagicianTest {

	public static void main(String[] args) {
		ApplicationContext ctx = new ClassPathXmlApplicationContext(
		"com/springinaction/aop/aop1.xml");
		Thinker thinker = (Thinker) ctx.getBean("volunteer");
		thinker.thinkOfSomething("你猜我在想什么？");
	}

}
```

**输出结果**

```
让我猜猜你在想什么？
你在想：你猜我在想什么？
```

---

## 注解引入
**注解引入所在切面**

```java
package com.springinaction.aop;

import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.DeclareParents;
@Aspect
public class Audience {
	@DeclareParents(
			value = "com.springinaction.aop.Performer+",
			defaultImpl = GraciousAssist.class)
	public static AssistPerformer assistPerformer;

}
```

**测试类**

```java
package com.springinaction.aop;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class AudienceTest {
	public static void main(String[] args) {
		ApplicationContext ctx = new ClassPathXmlApplicationContext(
		"com/springinaction/aop/aop1.xml");
		AssistPerformer ap = (AssistPerformer) ctx.getBean("drama");
		ap.assist();
	}
}
```

**输出结果**

```
GraciousAssist来助演了——
```
