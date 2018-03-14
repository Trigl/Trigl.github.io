---
layout:     post
title:      "简化 Spring XML 配置（二）"
subtitle:   "使用注解装配 bean"
date:       2016-01-10 00:00:00
author:     "Ink Bai"
catalog:    true
header-img: "http://ox2ru2icv.bkt.clouddn.com/image/post/post-bg-spring.jpg"
tags:
    - Spring
---

> 从Spring2.5开始，Spring开始支持使用注解的方式来自动装配bean的属性。这种方式与XML里面配置方式相比，减少了代码量，更加方便快捷。

如果想用注解来配置bean，首先要做的就是在XML文件中添加  `<context:annotation-config>`元素，在XML开始添加如下代码：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xmlns:context="http://www.springframework.org/schema/context"
xsi:schemaLocation="http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
http://www.springframework.org/schema/context
http://www.springframework.org/schema/context/spring-context-3.0.xsd">
<context:annotation-config />
<!-- bean declarations go here -->
</beans>
```

`<context:annotation-config>`会告诉Spring整个环境上下文，允许你使用基于注解的方式装配bean，从而Spring会自动将值注入属性、方法和构造器中。

Spring3提供了多种用于自动装配的注解方法：

 - Spirng自己的 @Autowired 注解
 - JSR-330的  @Inject 注解
 - JSR-250的 @Resource 注解

首先讲Spring的 @Autowired 注解方法。
## Spring的注解方式 @Autowired
#### 注入属性、方法、构造器

1.可以直接注入用@Autowired注入属性，不用管setter方法，即使是private类型的

  ```java
  @Autowired
  private Song songJay;
  ```

2.也可以直接在方法上加@Autowired注解，而且这个方法不再仅限于setter方法，所有注入属性的方法都可以使用@Autowired：

  ```java
  @Autowired
  public void setSongJay(Song songJay) {
    this.songJay = songJay;
  }
  ```

  或者

  ```java
	@Autowired
	public void playAgain(Song songJay) {
    this.songJay = songJay;
	}
  ```

  注意这里指的是所有注入属性的方法，即一个类自身的数据成员才可以注入，对于一般的方法参数，@Autowired是无法注入的，如下面这样就无法注入：

  ```java
	@Autowired
	public void playAgain(Song songJay) {
    songJay.play();
	}
  ```

3.也可以使用@Autowired给构造器加注解，但是注意加了自动注解以后在XML中就不可以再手动配置了，要用注解方式就只能使用注解方式：

  ```java
  @Autowired
  public SongJavaer(Song song) {
    this.song = song;
  }
  ```

#### 无法找到要注入的bean的情况
默认情况下，当使用注解的方式却没有找到符合条件的bean的时候，就会抛出NoSuchBeanDefinitionException异常，这并不是我们需要的，我们更希望没有找到的时候可以返回一个null，为了达到这样的目的，可以使用@Autowired的required属性：

```java
@Autowired(required=false)
private Song song;
```

将required属性设置为false就会在没有需要的bean的时候返回一个空而不是异常，注意当对构造器注解的时候，只有一个构造器可以将required设置为true，其他的都必须设置为false。并且，如果多个构造器都加了注解，最后会选择满足注入的参数最多的那个构造器。
#### 找到多个可选bean的情况
除了找不到bean，还有一种情况是找到很多个满足条件的bean，这种情况下也会抛出一个NoSuchBeanDefinitionException异常，为了帮助 @Autowired识别出哪一个bean才是你想要的，可以使用Spring的  @Qualifier注解

例如，找到的很多个bean中其中一个bean的id是guitar，那么就可以使用如下的方式挑选出这个bean：

```java
@Autowired
@Qualifier("songJay")
private Song song;
```

这里，@Qualifier括号里的内容是一个bean的id，事实上，这只是@Qualifier使用的特殊形式，我们一般是通过以下方式来定义可以让@Qualifier挑选的bean：

```xml
<bean class="com.springinaction.springidol.Guitar">
  <qualifier value="songJay" />
</bean>
```

这是在XML中配置`<qualifier>`元素的方式，也可以直接在类上加注解的方式，这样就不用在XML中配置了：

```java
@Qualifier("songJay")
public class Guitar implements Instrument {
  ...
}
```

#### 使用 @Qualifier创造自定义注解
除了使用@Qualifier注解以外，我们还可以通过@Qualifier来自己创造注解并且替换@Qualifier而使用自己的注解，如我们要创造一个@JaySong的注解，方式如下：

```java
package com.springinaction.springidol;

import java.lang.annotation.Retention;
import java.lang.annotation.Target;
import java.lang.annotation.ElementType;
import java.lang.annotation.RetentionPolicy;
import org.springframework.beans.factory.annotation.Qualifier;

@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface JaySong {
}
```

然后给被注解的和注解的类分别加上@JaySong即可

```java
@JaySong
public class SongJay implements Song {
  ...
}
```

```java
@Autowired
@JaySong
private Song songJay;
```

---

## Java的标准注解方式@Inject
为了统一各种依赖注入框架，JCP（Java Community Process）发布了依赖注入的Java标准：JSR-330，其中@Inject就是JSR-330的核心组件。

@Inject注解方式是Java自身的标准注解方式，但是Spring3以后对其提供支持。@Inject的注解几乎与Spring的@Autowired方式完全相同，都可以提供对属性、方法和构造器的注解，只有有限的几个方面稍有不同。
#### @Inject没有required属性
Spring的@Autowired是对required支持的，在没有找到所依赖bean的情况会返回一个null，但是@Inject却没有required属性，这就意味着必须有符合条件的bean，否则返回异常。
#### @Inject的Provider接口
对于Spring的bean来说一般都是单实例的，所以我们一般都是引用一个bean，但是如果我们现在需要引用一组相同类型的bean该怎么办呢？

第一种方法是定义好几个bean，这些bean都是相同的类，但是这仍然是单实例的。第二种方法不需要很多bean，而是采用@Inject提供的Provider接口，在构造方法中产生一个bean的很多个不同的实例。在类中的调用方式如下：

```java
	private Set<Song> songSet;
	@Inject
	public SongJavaer4(Provider<Song> songProvider) {
		songSet = new HashSet<Song>();
		for (int i = 0; i < 5; i++) {
			songSet.add(songProvider.get());
		}
	}
```

下面举一个实例来详细说明：重复播放一首歌五遍。

**首先给出主体类SongJavaer4**

```java
package com.springinaction.springidol;

import java.util.HashSet;
import java.util.Set;
import javax.inject.Inject;
import javax.inject.Provider;

public class SongJavaer4 implements Coder{
	private Set<Song> songSet;
	@Inject
	public SongJavaer4(Provider<Song> songProvider) {
		songSet = new HashSet<Song>();
		for (int i = 0; i < 5; i++) {
			songSet.add(songProvider.get());
		}
	}
	public void perform() {
		System.out.println("播放五遍——");
		for(Song song : songSet) {
			song.play();
		}
	}
}
```

**所要注入的依赖实体类Song1**

```java
package com.springinaction.springidol;

public class Song1 implements Song {
	public void play() {
		System.out.println("是他就是他，是他就是他，我们的英雄，小哪吒！......");
		System.out.println();
	}
}
```

**然后再配置XML**

```xml
	<bean class="com.springinaction.springidol.Song1" scope="prototype">
	</bean>

	<bean id="songJavaer4"
		class="com.springinaction.springidol.SongJavaer4">
	</bean>
```

注意配置依赖bean的时候注意加上scope="prototype"，这样就不再是单实例，而是每次注入依赖都产生一个新的实例。

**测试主方法**

```java
public class MainTest {
	public static void main(String[] args) {
		ApplicationContext ctx = new ClassPathXmlApplicationContext(
		"com/springinaction/springidol/spring-idol1.xml");
		Coder coder = (Coder) ctx.getBean("songJavaer4");
		coder.perform();
	}
}
```

控制台输出结果为：

```
播放五遍——
是他就是他，是他就是他，我们的英雄，小哪吒！......

是他就是他，是他就是他，我们的英雄，小哪吒！......

是他就是他，是他就是他，我们的英雄，小哪吒！......

是他就是他，是他就是他，我们的英雄，小哪吒！......

是他就是他，是他就是他，我们的英雄，小哪吒！......
```


####  @Inject的 @Qualifier注解
Spring的@Autowired注解通过@Qualifier来缩小查找bean范围，但是在Java的@Inject中是使用@Named来实现的，当然@Inject也有@Qualifier，但是@Inject的@Qualifier是不会直接使用，而是用来构造自定义注解的，如下：

```java
package com.springinaction.springidol;

import java.lang.annotation.Retention;
import java.lang.annotation.Target;
import java.lang.annotation.ElementType;
import java.lang.annotation.RetentionPolicy;

import javax.inject.Qualifier;


@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface JaySong {

}
```

其实可以发现，Java的@Inject通过@Qualifier创建自定义注解的方式和前面讲的Spring的@Autowired通过@Qualifier创建自定义注解的方式几乎完全一样，只不过@Inject引入的包是javax.inject.Qualifier，而@Autowired引入的包是org.springframework.beans.factory.annotation.Qualifier。
