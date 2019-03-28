---
layout:     post
title:      "Spring AOP（二）"
subtitle:   "在 XML 中配置切面"
date:       2016-03-16
author:     "Ink Bai"
catalog:    true
header-img: "/img/post/post-bg-spring.jpg"
tags:
    - Spring
---

> 在Spring AOP（一）中介绍了AOP的基本概念和几个术语，现在学习一下在XML中如何配置AOP。

在XML中AOP的配置元素有以下几种：

|AOP配置元素|描述|
|---|:---|
|`<aop:config>`|顶层的AOP配置元素，大多数的`<aop:*>`元素必须包含在`<aop:config>`元素内|
|`<aop:aspect>`|定义切面|
|`<aop:aspect-autoproxy>`|启用@AspectJ注解驱动的切面|
|`<aop:pointcut>`|定义切点|
|`<aop:advisor>`|定义AOP通知器|
|`<aop:before>`|定义AOP前置通知|
|`<aop:after>`|定义AOP后置通知（不管被通知的方法是否执行成功）|
|`<aop:after-returning>`|定义成功返回后的通知|
|`<aop:after-throwing>`|定义抛出异常后的通知|
|`<aop:around>`|定义AOP环绕通知|
|`<aop:declare-parents>`|为被通知的对象引入额外的接口，并透明地实现|


总之，个人理解AOP就是将切面的功能（通知），通过切点织入程序的执行过程中。下面是AOP术语的配置和关系图：

![这里写图片描述](/img/content/20160315232929169.jpg)

下面通过一个具体的栗子来学习，这个例子描述的是在一个正在表演的节目（相当于应用中一个正常的业务功能）中加入观众们的反响（相当于切面），要求在表演前观众们入座、手机关机或静音，表演中记录一下表演所用时间，表演后观众们喝彩或不满。

**表演类接口**

```java
package com.springinaction.aop;

public interface Performer {
	public void perform();
}
```

**表演类的一个实现：话剧演出**

```java
package com.springinaction.aop;

public class Drama implements Performer {

	@Override
	public void perform() {
		for (int i = 0; i < 1000; i++) {
			System.out.println("话剧正在进行中——");
		}
	}
}
```

**观众类作为切面**

```java
package com.springinaction.aop;

public class Audience {
	public void takeSeats() {
		// 节目开始之前
		System.out.println("演出前——观众开始入座");
	}

	public void turnOffCellPhones() {
		// 节目开始之前
		System.out.println("演出前——观众关机或静音");
	}

	public void applaud() {
		// 节目成功结束之后
		System.out.println("成功演出很成功——观众鼓掌：啪啪啪");
	}

	public void demandRefund() {
		// 节目表演失败之后
		System.out.println("节目演出很失败——切！一点都不好看，我们要求退钱！");
	}
}
```

## 准备工作

引入以下几个AOP包：

	>[aopalliance-1.0.jar](http://pan.baidu.com/s/1hryYVo4)
	[aspectjrt-1.7.4.jar](http://pan.baidu.com/s/1skfejZB)
	[aspectjweaver-1.7.4.jar](http://pan.baidu.com/s/1kUqL9f5)

在XML文件中加入包含关于Spring AOP命名空间的内容

	```xml
	<?xml version="1.0" encoding="UTF-8"?>
	<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:aop="http://www.springframework.org/schema/aop"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
	http://www.springframework.org/schema/aop
	http://www.springframework.org/schema/aop/spring-aop-3.0.xsd">

	</beans>
	```

---

## 定义切面
在XML中通过元素`<aop:config>`开启AOP配置，使用`<aop:aspect>`配置切面，如下：

```xml
     <aop:config>
        <aop:aspect ref="audience">
        </aop:aspect>
     </aop:config>
```

ref里面对应audience是切面bean的ID，也就是观众类对应的bean。观众类的bean和话剧表演的bean分别如下：

```xml
	<bean id="audience" class="com.springinaction.aop.Audience" />
	<bean id="drama" class="com.springinaction.aop.Drama" />
```

---

## 定义切点
#### AspectJ切点表达式语言
Spring AOP中使用的是AspectJ的切点表达式语言来定义的切点，前面讲过Spring AOP是基于代理的，而AspectJ的某些切点表达式是与基于代理的AOP无关的，因此Spring AOP支持的AspectJ切点指示器仅有有限的几种，如下表所示：

|AspectJ指示器|描述|
|---|---|
|execution{}|匹配到的连接点的执行方法<br>如：execution(* com.springinaction.aop.Performer.perform(..))<br>表达式表示当Performer的perform()方法执行时会触发通知， * 表明我们不关心方法返回值的类型， (..) 标识切点选择任意的paly()方法，无论该方法的参数是什么|
|args()|限制连接点匹配参数为指定类型的执行方法<br>如：execution(* com.springinaction.aop.Thinker.thinkOfSomething(String)) and args(thoughts)<br>注意到 thinkOfSomething(String) 有参数类型String，后面跟了一个 args(thoughts) ，这个是 thinkOfSomething() 方法要传给通知的参数，两个AspectJ知识器用and相连接，同样也可以使用or和not，当然我们可以使用&&、 &#124;&#124; 和 ! 来代替and、or和not|
|@args()|限制连接点匹配参数由制定注解标注的执行方法|
|this()|限制连接点匹配AOP代理的Bean引用为指定类型的类|
|target()|限制连接点匹配目标对象为制定类型的类|
|within()|限制连接点匹配特定的类型<br>如：execution(* com.springinaction.aop.Performer.perform(..)) && within(com.springinaction.aop.\*)<br> within(com.springinaction.aop.\*)表示com.springinaction.aop包下任意类的方法被调用时|
|@within()|限制连接点匹配指定注解所标注的类型（当使用Spring AOP时，方法定义在由指定的注解所标注的类里）|
|@annotation|限制匹配带有指定注解连接点|
|bean()|限制连接点只匹配特定的bean<br>如：execution(* com.springinaction.aop.Performer.perform(..)) adn bean(drama)<br>其中drama是一个类的beanID，这个bean是Performer接口的一个实现。加上了bean(drama)以后就只能匹配这个bean了|

上面的所有指示器中，execution是主要使用的，是必须的；其他的指示器都是限制所匹配的切点。另外，当Spring尝试使用除了上面的AspectJ的其他指示器时，将会抛出IllegalArgumentException异常。
#### 在XML中配置切点
使用元素`<aop:pointcut>`来配置切点，`<aop:pointcut>`既可以放在`<aop:aspect>`元素的作用域内，也可以放在`<aop:config>`元素的作用域下，对应的切点的作用范围就不同了。具体配置如下：

```xml
            <aop:pointcut
                id="performance"
                expression="execution(* com.springinaction.aop.Performer.perform(..))" />
```

id标识这个切点的id，expression里面就是上面讲到的AspectJ切点表达式语言。

---

## 定义通知
#### 声明前置和后置通知
分别使用`<aop:before>`和`<aop:after-returning>`以及`<aop:after-throwing>`元素来声明前置通知、返回结果后置通知和抛出异常后置通知，注意这几个元素是放在`<aop:aspect>`元素的作用域内。具体配置如下：

```xml
            <aop:before pointcut-ref="performance" method="takeSeats"/>
            <aop:before pointcut-ref="performance" method="turnOffCellPhones"/>
            <aop:after-returning pointcut-ref="performance" method="applaud"/>
            <aop:after-throwing pointcut-ref="performance" method="demandRefund"/>
```

该切面应用了4个不同的通知。两个`<aop:before>`元素定义了匹配切点的方法执行之前调用前置通知方法——audience Bean的takeSeats()和turnOffCellPhones()方法（由method属性所声明）。`<aop:after-returning>`元素定义了一个返回后（after-returning）通知，在切点所匹配的方法调用之后再执行applaud()方法。同样，`<aop:after-throwing>`元素定义了抛出后通知，如果所匹配的方式执行时抛出任何异常，都将调用demandRefund()方法。下图展示了通知逻辑如何编织到业务逻辑中。

![这里写图片描述](/img/content/20160316013140452.jpg)

下面我们用测试类来测试AOP实现

```java
package com.springinaction.aop;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class AudienceTest {
	public static void main(String[] args) {
		ApplicationContext ctx = new ClassPathXmlApplicationContext(
		"com/springinaction/aop/aop.xml");
		Performer drama = (Performer)ctx.getBean("drama");
		drama.perform();
	}
}
```

结果

```
演出前——观众开始入座
演出前——观众关机或静音
话剧正在进行中——
......（重复一千次）
成功演出很成功——观众鼓掌：啪啪啪
```

#### 声明环绕通知
前面的前置通知和后置通知分别发生在业务功能的前面和后面，如果想要实现跨越业务功能的整个事件段呢？例如表演节目除了进场关机和结束了鼓掌，我们还希望观众一直关注演出，报告演出的时间。如果通过前置和后置通知来实现的话，我们可以在一个成员变量中保存时间。但是由于Audience是单例，所以会存在线程安全问题。

环绕通知的作用在这里就体现出来了，它可以在一个方法中实现整个逻辑，即将切点的方法内嵌到环绕通知的方法中去。

**编写环绕通知方法**

下面是我们在观众类Audience中新添加的一个watchPerformance()方法作为AOP环绕通知。

```java
	public void watchPerformance(ProceedingJoinPoint joinpoint) {
		try {
			System.out.println("演出前——观众开始入座");
			System.out.println("演出前——观众关机或静音");
			long start = System.currentTimeMillis();

			joinpoint.proceed(); // 执行被通知的方法

			long end = System.currentTimeMillis();

			System.out.println("成功演出很成功——观众鼓掌：啪啪啪");
			System.out.println("演出持续了 " + (end - start) + " milliseconds");
		} catch (Throwable e) {
			System.out.println("节目演出很失败——切！一点都不好看，我们要求退钱！");
		}
	}
```

对于这个通知方法，注意它使用了ProceedingJoinPoint作为方法的入参。这个对象能让我们在通知里调用被通知方法。通知方法首先完成它需要做的事情，如果希望把控制权转给被通知的方法，我们可以调用ProceedingJoinPoint的proceed()方法。

**在XML声明环绕通知**

声明环绕通知与声明其他类型的通知没有太大的区别，我们需要做的仅仅是使用`<aop:around>`元素。

```xml
            <aop:around pointcut-ref="performance" method="watchPerformance"/>
```

然后我们通过测试类测试一下

```java
package com.springinaction.aop;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class AudienceTest {
	public static void main(String[] args) {
		ApplicationContext ctx = new ClassPathXmlApplicationContext(
		"com/springinaction/aop/aop.xml");
		Performer drama = (Performer)ctx.getBean("drama");
		drama.perform();
	}
}
```

输出结果：

```
演出前——观众开始入座
演出前——观众关机或静音
话剧正在进行中——
......（重复1000次）
演出很成功——观众鼓掌：啪啪啪
演出持续了 42 milliseconds
```

#### 为通知传递参数
上面的栗子通知和被通知的方法之间虽然实现了有了先后逻辑，但是并没有参数的传递，有时候我们需要校验传递给方法的参数值，这个时候为通知传递参数就非常有用了。

还是通过一个实例来说明：某位魔术师自称自己继承了其师傅刘半仙的衣钵，可以未卜先知知道他人内心所想；有一个志愿者不服，说要让他猜猜自己到底想了什么。这里就将魔术师作为切面。

**读心术接口**

```java
package com.springinaction.aop;

public interface MindReader {
	void interceptThoughts(String thoughts);

	String getThoughts();
}
```

**魔术师类实现读心术接口**

```java
package com.springinaction.aop;

public class Magician implements MindReader {
	private String thoughts;

	@Override
	public void interceptThoughts(String thoughts) {
		System.out.println("让我猜猜你在想什么？");
		this.thoughts = thoughts;
		System.out.println(this.getThoughts());
	}

	@Override
	public String getThoughts() {
		System.out.print("你在想：");
		return thoughts;
	}

}
```

**思考者接口**

```java
package com.springinaction.aop;

public interface Thinker {
	void thinkOfSomething(String thoughts);
}
```

**志愿者类实现思考者接口**

```java
package com.springinaction.aop;

public class Volunteer implements Thinker {
	private String thoughts;

	@Override
	public void thinkOfSomething(String thoughts) {
		this.thoughts = thoughts;
	}

	public String getThoughts() {
		return thoughts;
	}
}
```

**XML配置**

```xml
    <aop:config>
        <aop:aspect ref="magician">
            <aop:pointcut
                id="thinker"
                expression="execution(* com.springinaction.aop.Thinker.thinkOfSomething(String))
                               and args(thoughts)"/>

            <aop:before
                pointcut-ref="thinker"
                method="interceptThoughts"
                arg-names="thoughts"/>
        </aop:aspect>
    </aop:config>
```

从XML配置中我们可以看出，实现被通知方法向通知传参的关键有两点：

 1. 切点元素`<aop:pointcut>`的expression属性中使用了AspectJ切点表达式语言中的args()指示器，在本例中是args(thoughts)，传递的是thinkOfSomething(String thoughts)方法的参数。
 2. 通知元素`<aop:before>`中加入了 arg-names 属性，这个属性的值就是切点传过来的对应的参数thoughts。

**测试类**

```java
package com.springinaction.aop;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class MagicianTest {

	public static void main(String[] args) {
		ApplicationContext ctx = new ClassPathXmlApplicationContext(
		"com/springinaction/aop/aop.xml");
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

## 定义引入
#### 通过切面引入的原理
讲解引入之前先思考一下我们之前都做到了什么。我们已经通过AOP实现了为目标对象拥有的方法添加了新的功能，这是通过代理实现的，即切面为目标bean创建一个代理，通知发出的方法调用看似发往了目标bean，其实是被代理拦截了，即代理代替目标bean实现原有功能并添加了切面的新功能。

现在来说引入，什么是引入呢？就是在不改变切入的目标类的基础上为其添加新方法或属性。我们都知道Java不是动态语言，一旦类编译完成了，我们就很难再为该类添加新的功能了，那么所谓的“添加新方法或属性”是不是扯淡呢？是也不是，因为我们无法真的实现可以虚拟一个嘛，但你不说我不说谁知道呢，这就又用到了我们牛逼轰轰的代理。如下图所示：

![这里写图片描述](/img/content/20160316214542670.jpg)

这里，可以让代理发布一个新的接口，当引入接口的方法被调用时，代理就将此调用委托给了实现新接口的某个其他对象，也就是Bean的实现被拆分到了多个类，而不仅仅是原来的目标切点对应的那一个类。但是在外面看来，所有的功能都是通过切点对应的目标类的Bean实现的。

#### XML配置引入
还是通过一个实例来说明：前面的例子中表演类是一个切点，这里我们加入一个新的辅助表演类，来引入我们想要添加给表演类的方法。

**辅助表演接口**

```java
package com.springinaction.aop;

public interface AssistPerformer {
	void assist();
}
```

**助人为乐者实现辅助表演接口**

```java
package com.springinaction.aop;

public class GraciousAssist implements AssistPerformer {

	@Override
	public void assist() {
		System.out.println("GraciousAssist来助演了——");
	}

}
```

**XML配置**
引入是通过`<aop:declare-parents>`元素来实现的，这个元素在`<aop:aspect>`元素的作用域内，它声明了此切面所通知的Bean在它的对象层次结构中拥有新的父类，也就是它的代理拥有了新的父类。具体配置如下：

```xml
            <aop:declare-parents
                types-matching="com.springinaction.aop.Performer+"
                implement-interface="com.springinaction.aop.AssistPerformer"
                default-impl="com.springinaction.aop.GraciousAssist"/>
```

具体说明一下`<aop:declare-parents>`的属性：

 - types-matching：Performer接口所实现的子类，也就是对应的目标切点类。
 - implement-interface：新加入的接口类。
 - default-impl：新加入的接口的默认实现类。

除了使用default-impl来指定接口的实现外，还可以使用delegate-ref属性来标识：

```xml
            <aop:declare-parents
                types-matching="com.springinaction.aop.Performer+"
                implement-interface="com.springinaction.aop.AssistPerformer"
                delegate-ref="gracious"/>
```

其中“gracious”是新添加的接口的实现类对应Bean的ID：

```xml
    <bean id="gracious" class="com.springinaction.aop.GraciousAssist" />
```

**测试类**

```java
package com.springinaction.aop;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class AudienceTest {
	public static void main(String[] args) {
		ApplicationContext ctx = new ClassPathXmlApplicationContext(
		"com/springinaction/aop/aop.xml");
		Performer drama = (Performer)ctx.getBean("drama");
		drama.perform();
		AssistPerformer ap = (AssistPerformer) ctx.getBean("drama");
		ap.assist();
	}
}
```

注意一点，我们要实现的是新加入的AssistPerformer 接口的assist()方法，但是我们是通过切点的BeanID，也就是”drama“得到AssistPerformer 的实现的一个实例，这就仿佛是”drama“对应的切点Bean实现了AssistPerformer 接口一样，这就是AOP引入的神奇之处，可以为目标类添加新的方法。

**输出结果**

```
演出前——观众开始入座
演出前——观众关机或静音
话剧正在进行中——
......（重复1000次）
演出很成功——观众鼓掌：啪啪啪
演出持续了 42 milliseconds
GraciousAssist来助演了——
```
