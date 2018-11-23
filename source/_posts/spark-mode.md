---
layout:     post
title:      "Spark client mode 和 cluster mode 的区别"
date:       2018-04-28
author:     "Ink Bai"
catalog:    true
header-img: "/img/post/spark-mode.png"
tags:
    - Spark
---

在使用spark-submit提交Spark任务一般有以下参数：

```
./bin/spark-submit \
  --class <main-class> \
  --master <master-url> \
  --deploy-mode <deploy-mode> \
  --conf <key>=<value> \
  ... # other options
  <application-jar> \
  [application-arguments]
```

其中deploy-mode是针对集群而言的，是指集群部署的模式，根据Driver主进程放在哪分为两种方式：client和cluster，默认是client，下面我们就详细研究一下这两种模式的区别

## client mode

![这里写图片描述](/img/content/20170609165010221.png)

首先明白几个基本概念：Master节点就是你用来提交任务，即执行bin/spark-submit命令所在的那个节点；Driver进程就是开始执行你Spark程序的那个Main函数，虽然我这里边画的Driver进程在Master节点上，但注意Driver进程不一定在Master节点上，它可以在任何节点；Worker就是Slave节点，Executor进程必然在Worker节点上，用来进行实际的计算

1、client mode下Driver进程运行在Master节点上，不在Worker节点上，所以相对于参与实际计算的Worker集群而言，Driver就相当于是一个第三方的“client”

2、正由于Driver进程不在Worker节点上，所以其是独立的，不会消耗Worker集群的资源

3、client mode下Master和Worker节点必须处于同一片局域网内，因为Drive要和Executorr通信，例如Drive需要将Jar包通过Netty HTTP分发到Executor，Driver要给Executor分配任务等

4、client mode下没有监督重启机制，Driver进程如果挂了，需要额外的程序重启
## cluster mode
![这里写图片描述](/img/content/20170609183743824.png)

1、Driver程序在worker集群中某个节点，而非Master节点，但是这个节点由Master指定

2、Driver程序占据Worker的资源

3、cluster mode下Master可以使用--supervise对Driver进行监控，如果Driver挂了可以自动重启

4、cluster mode下Master节点和Worker节点一般不在同一局域网，因此就无法将Jar包分发到各个Worker，所以cluster mode要求必须提前把Jar包放到各个Worker几点对应的目录下面

## 总结
是选择client mode还是cluster mode呢？

一般来说，如果提交任务的节点（即Master）和Worker集群在同一个网络内，此时client mode比较合适。
如果提交任务的节点和Worker集群相隔比较远，就会采用cluster mode来最小化Driver和Executor之间的网络延迟。
## Refer
[官方文档](http://spark.apache.org/docs/latest/submitting-applications.html)
[Stackoverflow 的解答  ](https://stackoverflow.com/questions/37027732/spark-standalone-differences-between-client-and-cluster-deploy-modes)
[Quora 上的提问](https://www.quora.com/When-should-apache-spark-be-run-in-yarn-cluster-mode-vs-yarn-client-mode-A-use-case-example-for-both-approaches-would-be-more-helpful)
