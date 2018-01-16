---
layout:     post
title:      "简化 Spring XML 配置（三）"
subtitle:   "自动寻找 bean"
date:       2016-01-10 00:00:01
author:     "Ink Bai"
catalog:    true
header-img: "http://ox2ru2icv.bkt.clouddn.com/image/post/post-bg-spring.jpg"
tags:
    - Spring
---

讲解之前首先了解Spring从配置上下文环境到最后注入bean的整个过程

1. 首先是Spring加载上下文环境，即将所有的bean都放入容器中

  ```java
  ApplicationContext ctx = new ClassPathXmlApplicationContext("com/springinaction/springidol/spring-idol1.xml");
  Coder coder = (Coder) ctx.getBean("javaer4");
  ```

2. 然后从上下文环境中寻找加载的bean，一般来说我们已经在XML文件中通过配置`<bean>`配置好了需要寻找的bean。

  ```xml
  <bean id="song1" class="com.springinaction.springidol.Song1" />
  ```

3. 然后注入寻找到的bean依赖，通过前面讲的@Autowired或者@Inject

我们前文中其实讲的都是第三步，即注入bean时的一些简化XML的方法，现在我们讲一下第二步，即如何简化寻找bean的配置方法。想要寻找bean，必先定义bean，一般我们都是提前在XML文件中配置好了bean，现在如果我们不在XML中配置bean，而是在需要成为bean的那个类上加上某种注解，然后在XML中只配置一个范围，告诉容器在这个范围内找我们需要的bean，这样是不是简单很多了呢？

前文在讲自动注入的时候，在XML中加了`<context:annotation-config>`来使用自动注入功能，现在我们使用另一个元素 `<context:component-scan>`，这个元素首先能实现`<context:annotation-config>`的所有功能，更重要的是它是为了让Spring上下文自动寻找这个元素限定的范围中的bean并且自动声明bean，而不需要再使用`<bean>`来配置。

具体加入XML文件中的代码如下：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xmlns:context="http://www.springframework.org/schema/context"
xsi:schemaLocation="http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
http://www.springframework.org/schema/context
http://www.springframework.org/schema/context/spring-context-3.0.xsd">
	<context:component-scan
		base-package="com.springinaction.springidol">
	</context:component-scan>
```

`<context:component-scan>`元素工作原理是搜索一个包，寻找其中可以自动注解成Spring容器中的bean的类； base-package属性告诉`<context:component-scan>`应当从那个包开始寻找。

那么，`<context:component-scan>`是如何知道哪些类能成为Spring容器的bean呢？下面揭晓答案。

## 用于自动搜索的注解bean方式
`<context:component-scan>`搜索bean的默认方式是搜索被以下几种注解方式注解的类。

 - @Component——这是一种通用的注解方式，指明该类是一个Spring组件
 - @Controller——指明该类定义了一个SpringMVC的控制器
 - @Repository——指明该类定义了一个数据仓库
 - @Service——指明该类定义了一个服务
 - 任何使用了自定义注入的类必须使用@Component方式注解。

还是通过一个具体的实例来说明：程序员在听歌。

**首先给出主体类SongJavaer2**

```java
@Component("javaer")
public class SongJavaer2 implements Coder{
	@Inject
	private Song songJay;

	public void setSongJay(Song songJay) {
		this.songJay = songJay;
	}

	public void perform() {
		System.out.println("Trigl is writing " + " blogs.");
		System.out.println("While listening...");
		songJay.play();
	}
}
```

注意这里的注解方式@Component("javaer")，括号里面的内容相当于bean的名字，即在XML中配置bean时候的id。

**然后给出要注入的依赖类Song1**

```java
@Component
public class Song1 implements Song {
	public void play() {
		System.out.println("是他就是他，是他就是他，我们的英雄，小哪吒！......");
		System.out.println();
	}
}
```

这里的注解@Component没有括号中的内容，此时bean的id是默认名字，即对应首字母小写的类名，如该类名是Song1，那么默认bean名字就是song1.

**XML文件配置**

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xmlns:context="http://www.springframework.org/schema/context"
xsi:schemaLocation="http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
http://www.springframework.org/schema/context
http://www.springframework.org/schema/context/spring-context-3.0.xsd">
	<context:component-scan
		base-package="com.springinaction.springidol">
	</context:component-scan>
<!-- Bean declarations go here -->

</beans>
```

**main主函数进行测试**

```java
public class MainTest {
	public static void main(String[] args) {
		ApplicationContext ctx = new ClassPathXmlApplicationContext(
		"com/springinaction/springidol/spring-idol1.xml");
		Coder coder = (Coder) ctx.getBean("javaer");
		coder.perform();
	}
}
```

控制台输出为：

```
Trigl is writing  blogs.
While listening...
是他就是他，是他就是他，我们的英雄，小哪吒！......
```

---

## 过滤component-scans
使用`<context:component-scan>`已经很方便了，然而我们还是需要在类上加上@Component才可以，那能否更简化一下呢，例如不用在类上加@Component也可以找到？答案是可以的，但是仅限于实现一个接口的哪些类，通过添加 `<context:include-filter>`和`<context:exclude-filter>`就可以实现将某些继承于同一方法的类自动注解（取消自动注解）。

例如上面例子的很多类都是实现的Song接口，我们只要用如下方式就可以将其自动注解，而不需要在类上再加@Component

```xml
	<context:component-scan
		base-package="com.springinaction.springidol">
		<context:include-filter type="assignable"
			expression="com.springinaction.springidol.Song"/>
	</context:component-scan>
```

`<context:include-filter>`的type和expression属性共同为了自动注解某一接口的实现类而起作用。在上面定义的情况下，所有实现Song接口的类都将被自动注解成为Spring的bean，bean的id就是默认名字。

除了这种情况，type属性的其他取值如下：

|过滤类型|描述|
|------|------|
|anotation|默认的过滤方式，必须在类上加上注解，然后过滤器才能搜索到|
|assignable|过滤器搜索expression属性指定的接口的实现类，在这些实现类上不需要加注解|
|aspectj|过滤器搜索与expression属性指定的AspectJ表达式相匹配的类，在这些匹配的类上不需要加注解|
|custom|使用自定义的org.springframework.core.type.TypeFilter的实现类，该类由expression属性指定|
|regex|过滤器扫描类的名称与expression属性指定的正则表达式相匹配的那些类|

还是举一个实例来说明：程序员在听歌

**主体类SongJavaer2**

```java
@Component("javaer")
public class SongJavaer2 implements Coder{
	@Inject
	@Named("song1")
	private Song songJay;

	public void setSongJay(Song songJay) {
		this.songJay = songJay;
	}

	public void perform() {
		System.out.println("Trigl is writing " + " blogs.");
		System.out.println("While listening...");
		songJay.play();
	}
}
```

@Named("song1")指明依赖的bean，因为实现Song接口的bean有很多

**依赖类Song1**

```java
public class Song1 implements Song {
	public void play() {
		System.out.println("是他就是他，是他就是他，我们的英雄，小哪吒！......");
		System.out.println();
	}
}
```

这个类实现了Song接口，在下面的XML中将会配置，所以这里就不需要加上注解@Component，自动会声明一个bean，bean的id是默认的song1

**XML配置扫描过滤器**

```xml
	<context:component-scan
		base-package="com.springinaction.springidol">
		<context:include-filter type="assignable"
			expression="com.springinaction.springidol.Song"/>
	</context:component-scan>
```

**main主方法进行测试**

```java
public class MainTest {
	public static void main(String[] args) {
		ApplicationContext ctx = new ClassPathXmlApplicationContext(
		"com/springinaction/springidol/spring-idol1.xml");
		Coder coder = (Coder) ctx.getBean("javaer");
		coder.perform();
	}
}
```

控制台输出结果为：

```
Trigl is writing  blogs.
While listening...
是他就是他，是他就是他，我们的英雄，小哪吒！......
```
