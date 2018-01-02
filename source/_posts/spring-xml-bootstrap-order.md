---
layout:     post
title:      "关于 web.xml 中 Spring 配置文件的启动顺序"
subtitle:   "《Spring in Action》笔记，总体介绍一下Spring"
date:       2016-07-31
author:     "Ink"
catalog:    false
header-img: "http://ox2ru2icv.bkt.clouddn.com/image/post/post-bg-spring.jpg"
tags:
    - Spring
---

> 但凡搞 JavaWeb 开发的对 web.xml 应该是非常熟悉了，但是你真的了解其中的配置吗，这里我们讲一下 Spring 相关配置启动的先后顺序。

在web.xml中定义的Spring的配置文件一般有两个：

1、Spring上下文环境的配置文件：applicationContext.xml

```xml
	<context-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>
			classpath:applicationContext.xml
		</param-value>
	</context-param>
```

2、SpringMVC配置文件：spring-servlet.xml

```xml
	<servlet>
		<servlet-name>spring</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>classpath:spring-servlet.xml</param-value>
		</init-param>
		<load-on-startup>1</load-on-startup>
	</servlet>
```

加载顺序是：**首先加载Spring上下文环境配置文件，然后加载SpringMVC配置文件，并且如果配置了相同的内容，SpringMVC配置文件会被优先使用。**

所以这里需要注意一个问题，一定要注意SpringMVC配置文件内容不要把Spring上下文环境配置文件内容覆盖掉。

比如在Spring上下文环境配置文件中先引入service层，然后又加入了事务：

```xml
	<context:component-scan base-package="com.acms.service"></context:component-scan>
	<!-- define the transaction manager -->
	<bean id="transactionManagerOracle"
		class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
		<property name="dataSource" ref="dataSourceOracle" />
	</bean>

	<tx:annotation-driven transaction-manager="transactionManagerOracle" />
```

但是在SpringMVC配置文件中却默认引入所有类（当然也包括service层），但是没有加入事务

```xml
	<context:component-scan base-package="com.acms"></context:component-scan>
```

那么这时事务功能是无法起作用的，也就是代码中加入@Transactional注解是无效的。

所以为了防止这种问题一般是在Spring上下文配置文件中引入所有的类，并且加上事务：

```xml
	<context:component-scan base-package="com.acms"></context:component-scan>
	<!-- define the transaction manager -->
	<bean id="transactionManagerOracle"
		class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
		<property name="dataSource" ref="dataSourceOracle" />
	</bean>

	<tx:annotation-driven transaction-manager="transactionManagerOracle" />
```

而在SpringMVC配置文件中只引入controller层：

```xml
	<context:component-scan base-package="com.acms.controller" />
	<context:component-scan base-package="com.acms.*.controller" />
```
