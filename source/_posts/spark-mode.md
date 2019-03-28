---
layout:     post
title:      "Spark client mode 和 cluster mode 的区别"
date:       2018-04-28
author:     "Ink Bai"
catalog:    true
header-img: "/img/post/spark-mode.jpg"
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

![这里写图片描述](/img/content/20170609165010221.jpg)

首先明白几个基本概念：Master节点就是你用来提交任务，即执行bin/spark-submit命令所在的那个节点；Driver进程就是开始执行你Spark程序的那个Main函数，虽然我这里边画的Driver进程在Master节点上，但注意Driver进程不一定在Master节点上，它可以在任何节点；Worker就是Slave节点，Executor进程必然在Worker节点上，用来进行实际的计算

1、client mode下Driver进程运行在Master节点上，不在Worker节点上，所以相对于参与实际计算的Worker集群而言，Driver就相当于是一个第三方的“client”

2、正由于Driver进程不在Worker节点上，所以其是独立的，不会消耗Worker集群的资源

3、client mode下Master和Worker节点必须处于同一片局域网内，因为Drive要和Executorr通信，例如Drive需要将Jar包通过Netty HTTP分发到Executor，Driver要给Executor分配任务等

4、client mode下没有监督重启机制，Driver进程如果挂了，需要额外的程序重启
## cluster mode
![这里写图片描述](/img/content/20170609183743824.jpg)

1、Driver程序在worker集群中某个节点，而非Master节点，但是这个节点由Master指定

2、Driver程序占据Worker的资源

3、cluster mode下Master可以使用--supervise对Driver进行监控，如果Driver挂了可以自动重启

4、cluster mode下Master节点和Worker节点一般不在同一局域网，因此就无法将Jar包分发到各个Worker，所以cluster mode要求必须提前把Jar包放到各个Worker几点对应的目录下面

## Yarn-Client 和 Yarn-Cluster
#### Yarn-Client

![这里写图片描述](/img/content/yarn-client.jpg)

1. Spark Yarn Client向YARN的ResourceManager申请启动Application Master。同时在SparkContent初始化中将创建DAGScheduler和TASKScheduler等，由于我们选择的是Yarn-Client模式，程序会选择YarnClientClusterScheduler和YarnClientSchedulerBackend
2. ResourceManager收到请求后，在集群中选择一个NodeManager，为该应用程序分配第一个Container，要求它在这个Container中启动应用程序的ApplicationMaster，与YARN-Cluster区别的是在该ApplicationMaster不运行SparkContext，只与SparkContext进行联系进行资源的分派
3. Client中的SparkContext初始化完毕后，与ApplicationMaster建立通讯，向ResourceManager注册，根据任务信息向ResourceManager申请资源（Container）
4. 一旦ApplicationMaster申请到资源（也就是Container）后，便与对应的NodeManager通信，要求它在获得的Container中启动CoarseGrainedExecutorBackend，CoarseGrainedExecutorBackend启动后会向Client中的SparkContext注册并申请Task
5. client中的SparkContext分配Task给CoarseGrainedExecutorBackend执行，CoarseGrainedExecutorBackend运行Task并向Driver汇报运行的状态和进度，以让Client随时掌握各个任务的运行状态，从而可以在任务失败时重新启动任务
6. 应用程序运行完成后，Client的SparkContext向ResourceManager申请注销并关闭自己

因为是与Client端通信，所以Client不能关闭。
 客户端的Driver将应用提交给Yarn后，Yarn会先后启动ApplicationMaster和executor，另外ApplicationMaster和executor都 是装载在container里运行，container默认的内存是1G，ApplicationMaster分配的内存是driver-memory，executor分配的内存是executor-memory。同时，因为Driver在客户端，所以程序的运行结果可以在客户端显 示，Driver以进程名为SparkSubmit的形式存在。

 #### Yarn-Cluster
在YARN-Cluster模式中，当用户向YARN中提交一个应用程序后，YARN将分两个阶段运行该应用程序：

1. 第一个阶段是把Spark的Driver作为一个ApplicationMaster在YARN集群中先启动；
2. 第二个阶段是由ApplicationMaster创建应用程序，然后为它向ResourceManager申请资源，并启动Executor来运行Task，同时监控它的整个运行过程，直到运行完成，应用的运行结果不能在客户端显示（可以在history server中查看），所以最好将结果保存在HDFS而非stdout输出，客户端的终端显示的是作为YARN的job的简单运行状况

下图是yarn-cluster模式：

![这里写图片描述](/img/content/yarn-cluster1.jpg)
![这里写图片描述](/img/content/yarn-cluster2.jpg)

执行过程：

1. Spark Yarn Client向YARN中提交应用程序，包括ApplicationMaster程序、启动ApplicationMaster的命令、需要在Executor中运行的程序等
2. ResourceManager收到请求后，在集群中选择一个NodeManager，为该应用程序分配第一个Container，要求它在这个Container中启动应用程序的ApplicationMaster，其中ApplicationMaster进行SparkContext等的初始化
3. ApplicationMaster向ResourceManager注册，这样用户可以直接通过ResourceManage查看应用程序的运行状态，然后它将采用轮询的方式通过RPC协议为各个任务申请资源，并监控它们的运行状态直到运行结束
4. 一旦ApplicationMaster申请到资源（也就是Container）后，便与对应的NodeManager通信，要求它在获得的Container中启动CoarseGrainedExecutorBackend，而Executor对象的创建及维护是由CoarseGrainedExecutorBackend负责的，CoarseGrainedExecutorBackend启动后会向ApplicationMaster中的SparkContext注册并申请Task。这一点和Standalone模式一样，只不过SparkContext在Spark Application中初始化时，使用CoarseGrainedSchedulerBackend配合YarnClusterScheduler进行任务的调度，其中YarnClusterScheduler只是对TaskSchedulerImpl的一个简单包装，增加了对Executor的等待逻辑等
5. ApplicationMaster中的SparkContext分配Task给CoarseGrainedExecutorBackend执行，CoarseGrainedExecutorBackend运行Task并向ApplicationMaster汇报运行的状态和进度，以让ApplicationMaster随时掌握各个任务的运行状态，从而可以在任务失败时重新启动任务
6. 应用程序运行完成后，ApplicationMaster向ResourceManager申请注销并关闭自己

## 总结
用一句话来概括 client 和 cluster 模式的区别就是，driver 运行在集群中时，就是 cluster 模式，driver 运行在远离集群的某个地方，比如你本地机器上，就是 client 模式。

那么是选择client mode还是cluster mode呢？
一般来说，测试环境下会启动 client 模式，因为 client 端生成 driver，它需要和集群中的 executor 通信来调度它们的工作，因此 client 启动后不能断开，我们可以直接观测应用运行的情况，适合交互和调试。
而 cluster 下，当用户的 client 提交了作业以后，会在集群中生成 driver 进程，此时 client 就可以关闭了，并不会影响 spark 应用的正常运行，所以适合生产环境。
## Refer
[官方文档](http://spark.apache.org/docs/latest/submitting-applications.html)
[Stackoverflow 的解答  ](https://stackoverflow.com/questions/37027732/spark-standalone-differences-between-client-and-cluster-deploy-modes)
[Quora 上的提问](https://www.quora.com/When-should-apache-spark-be-run-in-yarn-cluster-mode-vs-yarn-client-mode-A-use-case-example-for-both-approaches-would-be-more-helpful)
[Spark On Yarn的两种模式yarn-cluster和yarn-client深度剖析](https://www.cnblogs.com/ITtangtang/p/7967386.html)
