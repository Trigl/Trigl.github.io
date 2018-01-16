---
layout:     post
title:      "Basic of Spring"
subtitle:   "装配 bean"
date:       2016-01-05
author:     "Ink Bai"
catalog:    true
header-img: "http://ox2ru2icv.bkt.clouddn.com/image/post/post-bg-spring.jpg"
tags:
    - Spring
---
>Spring bean 是基础中的基础，这里用一些生动的例子来讲解。

## 前言

在讲解Spring配置beans之前首先想一下一部成功的电影都需要哪些成员参与。首先，最重要的是要有导演、编剧、演员和投资人；其次，还有没那么明显的成员，音乐人、特技演员和艺术指导；此外，还有其他很重要但是容易被忽略的人，调音师、服装师、化妆师、宣传员、摄影师、摄影师助手、灯光指导和外卖小哥。一部成功的电影应当是将各个人员合理而且有序的组织起来，然后各自完成他们各自的工作，他们之间会有很多联系，大部分工作也需要很多人员共同完成。

对应到一个应用中，有很多个对象，这些对象之间只有相互发送消息才能实现功能，在原始的Java程序中，使各个对象产生联系的方法就是不停地new来new去，这就导致了代码复杂化，使其难以复用和实现单元测试。而在Spring里面，一个对象当需要用到另一个对象的时候，它并不负责找到这个对象并且创建它，相反，仅仅给出需要的对象的引用就可以了，具体的对象实例是放在container里面的。

将一个应用中不同对象联系起来的行为就是依赖注入（dependency injection）的本质，称之为装配（wiring），本文讲解的就是装配beans的方式，这些内容是Spring最常用到的。

---

## 声明bean
#### 在XML文件中配置Spring
前文已经说过，容器是Spring的基础，而我们将各个beans创建出来并且将它们联系起来是通过一个或多个XML文件的配置来实现的。当在XML中声明beans的时候，基本元素就是`<beans>`，一个标准的Spring配置的XML文件如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans-3.0.xsd">
<!-- Bean declarations go here -->
</beans>
```

在`<beans>`里面可以做Spring的相关配置，包括`<bean>`声明。但是`beans`并非Spring唯一的命名空间，Spring框架的核心总共有十个配置的命名空间，如下：

 - aop：为声明切面以及将用@AspectJ注解的类自动代理为Spring切面提供了配置元素
 - beans：声明和装配bean，是Spring最核心也是最原始的命名空间
 - context：用来配置Spring应用上下文环境的元素，包括自动检测和装配bean以及注入某些非Spring直接管理的对象的能力。
 - jee：提供了与Java EE API的集成，比如JNDI和EJB（原文：Offers integration with Java EE APIs such as JNDI and EJB.）
 - jms：为声明消息驱动的POJOs提供了配置元素（不是太懂，原文：Provides configuration elements for declaring message-driven POJOs.）
 - lang：可以声明由Groovy，JRuby或者BeanShell脚本实现的bean（原文：Enables declaration of beans that are implemented as Groovy, JRuby, or BeanShell scripts.）
 - mvc：可以使用MVC的能力，如面向注解的控制器、视图控制器和拦截器
 - oxm：提供Spring的对象到XML的映射
 - tx：提供声明的事物配置（原文：Provides for declarative transaction configuration.）
 - util：提供各种各样的工具类元素，包括把集合声明为bean、支持属性占位符元素（原文：A miscellaneous selection of utility elements. Includes the ability to declare collections as beans and support for property placeholder elements.）

#### 声明一个简单的bean
首先写一个非常实例：用Spring配置一个bean，这个bean指向的类里有一个方法，用来描述一下动作：java学习者在写博客。

**首先写出这个类：Javaer**

```java
package com.springinaction.springidol;

/**
 * Java程序员
 * @author 白鑫
 * @date 2016-1-5下午11:13:16
 */
public class Javaer implements Coder {
	private int blogNum = 2;//博客数
	public Javaer() {
	}
	public Javaer(int blogNum) {
		this.blogNum = blogNum;
	}
	public void perform() {
		System.out.println("Trigl is writing " +  blogNum + " blogs.");
		}
}
```

**继承的接口：Coder**

```java
package com.springinaction.springidol;

/**
 * Coder接口类
 * @author 白鑫
 * @date 2016-1-5下午11:15:27
 */
public interface Coder {
	void perform();
}
```

继承的目的是为了减藕，毋庸多言。

**XML配置bean：spring-idol.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans-3.0.xsd">
<!-- Bean declarations go here -->
	<bean id="javaer" class="com.springinaction.springidol.Javaer">
	</bean>
</beans>
```

`<bean>`是Spring中最基本的配置单元，它会告诉Spring为你创建一个对象。id属性是这个bean的名字，通过它指向Spring容器。class属性说明这个bean对应的类。

当Spring的容器加载bean的时候，就会使用默认的构造器实例化名为javaer的bean，但是本质上，javaer的创建就是通过以下的Java代码：

```java
new com.springinaction.springidol.Javaer();
```

**在main方法中加载加载上下文环境进行测试**

```java
	public static void main(String[] args) {
		ApplicationContext ctx = new ClassPathXmlApplicationContext(
		"com/springinaction/springidol/spring-idol.xml");
		Coder coder = (Coder) ctx.getBean("javaer");
		coder.perform();
	}
```

结果输出为：

```
Trigl is writing 2 blogs.
```

#### 在bean中自定义构造器参数
我们可以通过自定义一个构造器来创建一个对象，这个操作同样可以在Spring中实现，只要将构造器的参数写进XML的bean配置即可。

还是举一个实例来说明：用Spring配置一个bean，这个bean指向的类里有一个方法，用来描述一下动作：java学习者在写10篇博客同时还在听歌。（这里加入10和歌曲是为了在构造器中加入一个原始数据类型和一个类作为构造器参数）

**首先写出主体类：SongJavaer**

```java
package com.springinaction.springidol;

/**
 * 听歌的Java程序员
 * @author 白鑫
 * @date 2016-1-6上午12:29:11
 */
public class SongJavaer extends Javaer {
	private Song song;

	public SongJavaer(int blogNum, Song song) {
		super(blogNum);
		this.song = song;
	}

	public void perform() {
		super.perform();
		System.out.println("While listening...");
		song.play();
	}
}
```

由于这个例子与上个例子相比多了个听歌这个动作，所以我们新建一个SongJavaer类继承上面的Javaer类，可以看到与上面的例子相比多了个构造方法，这个构造方法的参数一个是int原始数据类型的，一个是类Song，在下面我会给出在XML文件中配置构造器参数的方法。

**Song接口类：Song**

```java
package com.springinaction.springidol;
/**
 * Song接口
 * @author 白鑫
 * @date 2016-1-6上午12:36:05
 */
public interface Song {
	void play();
}
```

**再写一个继承Song接口的类：SongJay**

```java
package com.springinaction.springidol;
/**
 * Jay的歌
 * @author 白鑫
 * @date 2016-1-6上午12:37:28
 */
public class SongJay implements Song {
	private static String[] LYRICS = {
		"一盏离愁孤灯伫立在窗口",
		"我在门后假装你人还没走",
		"旧地如重游月圆更寂寞",
		"夜半清醒的烛火不忍苛责我",
		"一壶漂泊浪迹天涯难入喉",
		"你走之后酒暖回忆思念瘦",
		"水向东流时间怎么偷",
		"花开就一次成熟我却错过",
		"谁在用琵琶弹奏一曲东风破",
		"岁月在墙上剥落看见小时候",
		"犹记得那年我们都还很年幼",
		"而如今琴声幽幽我的等候你没听过",
		"谁在用琵琶弹奏一曲东风破",
		"枫叶将故事染色结局我看透",
		"篱笆外的古道我牵着你走过",
		"荒烟漫草的年头就连分手都很沉默"
	};

	public void play() {
		for (int i = 0; i < LYRICS.length; i++){
			System.out.println(LYRICS[i]);
		}
	}
}
```

**在XML文件spring-idol.xml中配置bean**

```xml
	<bean id="songJavaer" class="com.springinaction.springidol.SongJavaer">
		<constructor-arg value="20" />
		<constructor-arg ref="songJay" />
	</bean>

	<bean id="songJay" class="com.springinaction.springidol.SongJay" />
```

可以看到，在`<bean>`中配置构造器参数是通过`<constructor-arg>`属性来配置的。其中对于简单的原始数据类型如int、String类型，是通过`<constructor-arg>`的value属性来传参的；对于具体的类，是通过使用`<constructor-arg>`的ref属性来传参的，ref中的值等于相应bean的id。

**在main方法中加载上下文环境进行测试**

```java
	public static void main(String[] args) {
		ApplicationContext ctx = new ClassPathXmlApplicationContext(
				"com/springinaction/springidol/spring-idol.xml");
		Coder coder = (Coder) ctx.getBean("songJavaer");
		coder.perform();
	}
```

控制台输出的结果是：

```
Trigl is writing 20 blogs.
While listening...
一盏离愁孤灯伫立在窗口
我在门后假装你人还没走
旧地如重游月圆更寂寞
夜半清醒的烛火不忍苛责我
一壶漂泊浪迹天涯难入喉
你走之后酒暖回忆思念瘦
水向东流时间怎么偷
花开就一次成熟我却错过
谁在用琵琶弹奏一曲东风破
岁月在墙上剥落看见小时候
犹记得那年我们都还很年幼
而如今琴声幽幽我的等候你没听过
谁在用琵琶弹奏一曲东风破
枫叶将故事染色结局我看透
篱笆外的古道我牵着你走过
荒烟漫草的年头就连分手都很沉默
```

**通过工厂方法创建单实例**

在Spring中可以将一个单实例类配置成为一个bean，如下面的Head

```java
package com.springinaction.springidol;

/**
 * 工厂方法实现单实例类
 * @author 白鑫
 * @date 2016-1-6下午08:46:01
 */
public class Head {
	private Head() {
	}
	private static class HeadSingletonHolder {
		static Head instance = new Head();
	}
	public static Head getInstance() {
		return HeadSingletonHolder.instance;
	}
}
```

在XML中的配置为：

```xml
	<bean id="theHead" class="com.springinaction.springidol.Head" factory-method="getInstance">
	</bean>
```

单实例类每次创建都是以static的形式，这就保证了每次创建的对象都是一样的。在Spring中的配置是通过`<bean>`元素中的factory-method属性来配置，里面的值对应于类中的一个static方法getInstance。注意factory-method不仅局限于将一个单实例配置成为bean，对于任何你想要编写一个通过static方法创造类的时候，都可以使用工厂方法。
#### Bean的作用域
所以的Spring的bean都默认是单实例的，即每次Spring分配一个bean以后，总是会产生完全相同的对象。有时候我们不需要单实例，而是需要每次都产生唯一的实例。比如说买票，必须保证每次创建票的bean时候都是唯一的，我们可以通过以下方式来配置：

```xml
<bean id="ticket" class="com.springinaction.springidol.Ticket" scope="prototype" />
```

也就是通过设置`<bean>`的scope属性值为prototype来保证每次加载的bean都会创造唯一的实例。

---

## 注入bean属性
上文中讲的是构造器注入的方式，现在讲如何注入bean的属性。bean的属性是什么，其实就是一个类对应的数据成员。当一个类定义了一个数据成员以后，一般就会有对应于这个数据成员的setXXX()和getXXX()方法，而Spring注入bean属性就是通过setter注入的。
#### 注入简单的值和引用
仍然先举一个例子：用Spring配置一个bean，给这个bean注入属性，这个bean指向的类里有一个方法，用来描述一下动作：java学习者在写30篇博客同时还在听歌。

**首先还是先写一个主体类：SongJavaer2**

```java
package com.springinaction.springidol;

/**
 * 听歌的编程者
 * @author 白鑫
 * @date 2016-1-6下午11:01:14
 */
public class SongJavaer2 implements Coder{
	private int blogNum;
	public void setBlogNum(int blogNum) {
		this.blogNum = blogNum;
	}

	private Song song;
	public void setSong(Song song) {
		this.song = song;
	}

	public void perform() {
		System.out.println("Trigl is writing " +  blogNum + " blogs.");
		System.out.println("While listening...");
		song.play();
	}
}
```

继承的接口Coder.java见上面，这个类定义了两个数据成员：int类型的blogNum和类Song的对象song，下面我们将实现用Spring将这两个属性注入。

**在XML文件中配置bean**

```xml
	<bean id="songJavaer2" class="com.springinaction.springidol.SongJavaer2">
		<property name="blogNum" value="30" />
		<property name="song" ref="songJay" />
	</bean>
	<bean id="songJay" class="com.springinaction.springidol.SongJay" />
```

可以看到，Spring注入属性是通过`<bean>`元素的`<property>`属性来实现的，其中name的值对应所注入的类的数据成员，也即bean的属性。当注入普通的原始数据成员类型如整型、字符型的时候，是用value来赋值的；当注入另一个bean的引用的时候，是用ref来赋值的。

**main方法加载上下文环境进行测试**

```java
	public static void main(String[] args) {
		ApplicationContext ctx = new ClassPathXmlApplicationContext(
		"com/springinaction/springidol/spring-idol.xml");
		Coder coder = (Coder) ctx.getBean("songJavaer2");
		coder.perform();
	}
```

控制台输出结果是：

```
Trigl is writing 30 blogs.
While listening...
一盏离愁孤灯伫立在窗口
我在门后假装你人还没走
旧地如重游月圆更寂寞
夜半清醒的烛火不忍苛责我
一壶漂泊浪迹天涯难入喉
你走之后酒暖回忆思念瘦
水向东流时间怎么偷
花开就一次成熟我却错过
谁在用琵琶弹奏一曲东风破
岁月在墙上剥落看见小时候
犹记得那年我们都还很年幼
而如今琴声幽幽我的等候你没听过
谁在用琵琶弹奏一曲东风破
枫叶将故事染色结局我看透
篱笆外的古道我牵着你走过
荒烟漫草的年头就连分手都很沉默
```

#### 注入内部bean
一般来说，我们注入属性时引用的bean还可以被其他bean所引用，但是如果我们想单独霸占一个bean，即仅被某一个bean所使用，其他bean无法使用，那该怎么办呢？很简单，直接在`<property>`属性里面写一个`<bean>`元素即可，但是注意，`<bean>`元素里面只有class没有id，当然你也可以写一个id，但是这样完全没有必要，因为这个bean不会被其他bean引用，所以定义了id也不可能用到，例子如下：

```xml
	<bean id="songJavaer2" class="com.springinaction.springidol.SongJavaer2">
		<property name="song">
			<bean class="org.springinaction.springidol.SongJavaer" />
		</property>
	</bean>
```

而且在构造器注入时也可以这样注入内部bean，如下：

```xml
	<bean id="songJavaer" class="com.springinaction.springidol.SongJavaer">
		<constructor-arg>
			<bean class="org.springinaction.springidol.SongJavaer" />
		</constructor-arg>
	</bean>
```

#### 注入集合
上面我们讲了注入简单的值和引用的bean，还有一种复杂的情况就是需要注入这些值或者bean的集合。一般在类中可以选择作为数据成员的集合性的类有以下几种：数组、java.util.Collection、java.util.Set、java.util.List、java.util.Map、java.util.Properties，对应于Spring在XML文件中配置bean的元素有`<list>`、`<set>`、`<map>`、`<props>`，具体差别见下表：

|集合配置元素|可注入的集合类|配置说明|
|:-----------|:---------------|:---------|
|`<list>`|数组、Collection、Set、List|允许重复的集合|
|`<set>`|数组、Collection、Set、List|不允许重复的集合|
|`<map>`|Map|对应的key和value可以取任何类型|
|`<props>`|Properties|对应的key和value只可以取String类型|

细节知识仍然是通过例子来讲解：用Spring配置一个bean，给这个bean注入属性，这个属性要求是集合类，这个bean指向的类里有一个方法，用来描述一下动作：java学习者在听一组歌曲。

**首先写主体类SongJavaer3**

```java
package com.springinaction.springidol;

import java.util.Collection;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.Properties;

/**
 * 唱歌的编程者
 * @author 白鑫
 * @date 2016-1-6下午11:29:49
 */
public class SongJavaer3 implements Coder {
	//Collection在XML中对应于<list>或者<set>
	private Collection<Song> songCollection;
	public void setSongCollection(Collection<Song> songCollection) {
		this.songCollection = songCollection;
	}

	//List在XML中对应于<list>或者<set>
	private List<Song> songList;
	public void setSongList(List<Song> songList) {
		this.songList = songList;
	}

	//Map在XML中对应于<map>
	private Map<String,Song> songMap;
	public void setSongMap(Map<String,Song> songMap) {
		this.songMap = songMap;
	}

	//Properties在XML中对应于<props>
	private Properties songProperties;
	public void setSongProperties(Properties songProperties) {
		this.songProperties = songProperties;
	}

	public void perform() {
		System.out.println("Trigl一边写Java博客一边听歌");
		System.out.println("————————我是可爱的分割线————————");
		System.out.println("播放songCollection中的歌曲......");
		System.out.println();
		for(Song song : songCollection) {
			song.play();
		}
		System.out.println("————————我是可爱的分割线————————");
		System.out.println("播放songList中的歌曲......");
		System.out.println();
		for(Song song : songList) {
			song.play();
		}
		System.out.println("————————我是可爱的分割线————————");
		System.out.println("播放songMap中的歌曲......");
		System.out.println();
		for(String key : songMap.keySet()) {
			System.out.print(key + "：");
			Song song = songMap.get(key);
			song.play();
		}
		System.out.println("————————我是可爱的分割线————————");
		System.out.println("播放songProperties中的歌曲......");
		System.out.println();
		for(Iterator it = songProperties.keySet().iterator();it.hasNext();)
		{
			String key = (String)it.next();//key值
			System.out.print(key + "：");
			System.out.println(songProperties.getProperty(key));//通过key得到value
			System.out.println();
		}
	}
}
```

这里写了四个例子，分别为了用于后面讲解四种集合配置元素的使用。

**Song1**

```java
package com.springinaction.springidol;

public class Song1 implements Song {
	public void play() {
		System.out.println("是他就是他，是他就是他，我们的英雄，小哪吒！......");
		System.out.println();
	}
}
```

**Song2**

```java
package com.springinaction.springidol;

public class Song2 implements Song {
	public void play() {
		System.out.println("让我们荡起双桨，小船儿随风飘荡~......");
		System.out.println();
	}
}
```

**Song3**

```java
package com.springinaction.springidol;

public class Song3 implements Song {
	public void play() {
		System.out.println("梦里梦到醒不来的梦，梦境里被囚禁的红......");
		System.out.println();
	}
}
```

**XML文件中集合注入元素的具体配置**

```xml
	<bean id="songJavaer3" class="com.springinaction.springidol.SongJavaer3">
		<!-- 通过<list>元素注入，可以显示重复的bean -->
		<property name="songCollection">
			<list>
				<ref bean="song1"/>
				<ref bean="song2"/>
				<ref bean="song3"/>
				<ref bean="song3"/>
			</list>
		</property>

		<!-- 通过<set>元素注入，不显示重复的bean -->
		<property name="songList">
			<set>
				<ref bean="song1"/>
				<ref bean="song1"/>
				<ref bean="song1"/>
			</set>
		</property>

		<!-- 通过<map>元素注入 -->
		<property name="songMap">
			<map>
				<entry key="《哪吒》" value-ref="song1" />
				<entry key="《让我们荡起双桨》" value-ref="song2" />
				<entry key="《红玫瑰》" value-ref="song3" />
			</map>
		</property>

		<!-- 通过<props>元素注入 -->
		<property name="songProperties">
			<props>
				<prop key="《哪吒》">是他就是他，是他就是他，我们的英雄，小哪吒！......</prop>
				<prop key="《让我们荡起双桨》">让我们荡起双桨，小船儿随风飘荡~......</prop>
				<prop key="《红玫瑰》">梦里梦到醒不来的梦，梦境里被囚禁的红......</prop>
			</props>
		</property>
	</bean>
	<bean id="song1" class="com.springinaction.springidol.Song1" />
	<bean id="song2" class="com.springinaction.springidol.Song2" />
	<bean id="song3" class="com.springinaction.springidol.Song3" />
```

`<map>`有四个属性值，如下：

|属性|目的|
|---|---|
|key|指明map的键，取String|
|key-ref|指明map的键，取bean的引用（id）|
|value|指明map的值，取String|
|value-ref|指明map的值，取bean的引用（id）|


**main方法中进行测试**

```java
package com.springinaction.springidol;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

/**
 * 用来测试的main函数
 * @author 白鑫
 * @date 2016-1-6下午11:36:08
 */
public class MainTest {
	public static void main(String[] args) {
		ApplicationContext ctx = new ClassPathXmlApplicationContext(
		"com/springinaction/springidol/spring-idol.xml");
		Coder coder = (Coder) ctx.getBean("songJavaer3");
		coder.perform();
	}
}
```

控制台输出为：

```
Trigl一边写Java博客一边听歌
————————我是可爱的分割线————————
播放songCollection中的歌曲......

是他就是他，是他就是他，我们的英雄，小哪吒！......

让我们荡起双桨，小船儿随风飘荡~......

梦里梦到醒不来的梦，梦境里被囚禁的红......

梦里梦到醒不来的梦，梦境里被囚禁的红......

————————我是可爱的分割线————————
播放songList中的歌曲......

是他就是他，是他就是他，我们的英雄，小哪吒！......

————————我是可爱的分割线————————
播放songMap中的歌曲......

《哪吒》：是他就是他，是他就是他，我们的英雄，小哪吒！......

《让我们荡起双桨》：让我们荡起双桨，小船儿随风飘荡~......

《红玫瑰》：梦里梦到醒不来的梦，梦境里被囚禁的红......

————————我是可爱的分割线————————
播放songProperties中的歌曲......

《让我们荡起双桨》：让我们荡起双桨，小船儿随风飘荡~......

《哪吒》：是他就是他，是他就是他，我们的英雄，小哪吒！......

《红玫瑰》：梦里梦到醒不来的梦，梦境里被囚禁的红......
```

#### 注入空
是的少年，你没有听错，的确是注入空或者说是null。第一个原因是为了初始化属性的值，另一个原因是覆盖自动装配的值，配置方式如下：

```xml
<property name="someNonNullProperty"><null/></property>
```
