---
layout:     post
title:      "系统炸了怎么办？"
date:       2020-10-15
author:     "Ink Bai"
catalog:    true
header-img: "/img/post/swiss-army.jpg"
tags:
    - 小叮当的口袋
---

> 线上出问题的时候，排查速度是放在第一位的，工欲善其事必先利其器，今天就来总结一下排查常用的工具和命令。

当遇到突发问题时，在我看来，正确的解决步骤应当按照下面几步进行：

- 评估影响
- 止损恢复
- 排查恢复
- case study

## 评估影响
某天突然来了告警，线上出问题，怎么办？先不要慌，千万不要一上来就想找到原因，因为你不一定能马上找到问题，当然如果问题特别简单或者你很快就找到了解决方案（5分钟内）的话另当别论。

首先应当做的是评估影响，具体包括：

**1、通过问题的具体现象，初步确认问题等级**
比如一个网站发现登录不了，那就是很严重的问题，几乎是 P0 级别，就需要立刻马上解决。而如果仅仅是网站博客的 PV 统计有问题，不会对用户造成特别恶劣的影响，那么问题的等级就不是很高，可以先把问题记录下来，后续排期解决即可。

**2、确认受影响用户或者业务方，故障时间，通知对方**
出了问题不可怕，可怕的是对方找上门来了。

如果线上出了问题，你在所有人知道之前就给解决了，那么此时内心一定有一种劫后余生的快感。

然而大多数情况不会那么好运，问题不会马上解决，企业微信里该疯狂@你还是会疯狂@你。

所以如果确认了问题以后，发现问题很严重，那么与其后知后觉地被别人找上门来怼，主动告知对方就很重要了，这样除了给对方一个心理预期，对方也可以进行协作，进行一些止损和恢复的操作。

通知的内容一般包括几个方面：故障现象、影响的业务、故障时间、可能的原因、预计恢复时间。

评估影响到哪些业务的方式有很多，如根据经验、日志，但最好是能够自动化，监控化，工具化，这样遇到突发问题才能更加镇定，不会手忙脚乱。比如你负责一个数据上报网关系统，目前现象是移动端的数据没有报上来，查看监控发现网关全部返回502了，那么初步判断只会对移动端的数据上报造成影响。

总之第一时间发现问题并且通知到受影响方是很重要的，如果你总是不能第一个发现自己的线上问题，而是业务方先发现的，那就要反思系统的监控告警是不是有不到位的地方了。
## 止损恢复
理论上出了问题以后第一时间是要进行止损和恢复的，上面的「评估影响」可以让其他同学并发去做。

止损恢复分为两步：

**1、保留现场**
恢复是第一步要进行的，但如果时间允许的话最好还是保留一下现场方便后续排错。排错就像判案，保留现场是非常重要的，毕竟巧妇难为无米之炊。

当然如果实在是情况紧急也可以不用保留现场直接快速恢复，待后续再想办法复现。

需要考虑保留的内容有：

- 日志，日志一般都会实时写入 ES，此时物理机上的就不用保留了
- 监控，一般都会上报到 prometheus，也不需要保留
- 线程栈：Java 应用的线程情况，后面讲如何导出
- 内存情况：Java 内存分析，后面细讲
- GC 日志
- 如果是分布式部署，可以考虑恢复大部分节点，只保留一个问题点后续排查

**2、恢复**
恢复的方法有很多，具体视情况而定：

- 最常见的，如果上线代码或者配置有 bug，立刻回滚
- 当然还有万能的重启大法
- 如果容量不够，那就考虑横向或者纵向扩容

## 排查修复
线上恢复之后就要开始找寻 root cause 了，这就需要使出十八般武艺了，注意排查问题千万不要一上来就去扣代码，君子性非异也善假于物也，使用强大的工具能让我们事半功倍。

**1、看监控**
- 业务监控，如 QPS、RT、错误数
- 基础监控，CPU、磁盘 IO、网络、连接数、内存、线程数、GC 情况等。
- 链路调用情况 dapper

**2、查日志**
找到短时间内大量出现的错误日志，看有何异常。

**3、Java 排查工具**
jps：快速定位对应的 Java 进程 ID。

jmap：输出某个 java 进程内存情况。
```
jmap -heap pid   输出当前进程 JVM 堆新生代、老年代、持久代等请情况，GC 使用的算法等信息
jmap -histo:live {pid} | head -n 10  输出当前进程内存中所有对象包含的大小
jcmd pid GC.heap_dump /gc/dump.hprof 以二进制输出档当前内存的堆情况，然后可以导入 MAT 等工具进行
```

jstack: 打印某个 Java 线程的线程栈信息
```
jstack pid > file
```

jstat：实时打印 GC 情况
```
jstat -gcutil pid 5000
```

Java 开启远程 debug 的 JVM 参数：
```
-Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=5010
```

Java 问题排查 shell 脚本：[Java相关脚本](https://github.com/oldratlee/useful-scripts/blob/master/docs/java.md)

java飞行记录器JFR，分析程序瓶颈：[Jvm 性能分析工具-JMC+JFR 小记](https://www.jianshu.com/p/a4d98b1dc3b7)

火焰图，分析 cpu 使用：[java性能火焰图的生成](https://www.cnblogs.com/hama1993/p/10580581.html)
## case study
问题解决之后一定要回顾总结，避免下次再犯。线上出问题表面上看会有各种原因，但本质上还是反映出系统本身存在的问题，复盘就是让我们找出系统层面存在的本质问题，总结经验并且做出优化。

另外，阿里云有一篇文章讲的很好，推荐阅读：[救火必备！问题排查与系统优化手册](https://developer.aliyun.com/article/767550)