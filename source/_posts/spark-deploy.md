---
layout:     post
title:      "Spark 部署要点"
date:       2018-10-24
author:     "Ink Bai"
catalog:    true
header-img: "/img/post/spark-deploy.jpg"
tags:
    - Spark
---
> 写完一个 Spark 应用程序以后，如何把它部署到集群上运行呢，这个部署的过程是怎样的呢，这里我主要以 Yarn 集群为主来讲解。

## 基本术语
让我们先了解一下 Spark 中常用的一些基本术语：

术语|含义
-|-
Application|用户创建的 Spark 应用，会在集群中形成 driver 进程和 executor 进程。
Application jar|包含用户 Spark 应用和其相关依赖的 Jar 包，不应当包含 Spark 或者 Hadoop 的依赖，因为运行时集群会提供。
Driver program|运行 Spark 应用 `main()` 函数并且创建 SparkContext 的进程。
Cluster manager|用来为 Spark 应用分配资源的外部服务（如 standalone，Mesos 和 Yarn）。
Deploy mode|部署模式，不同部署模式的区别就是 driver 进程运行在哪里，具体区别看这篇文章：[Spark client mode 和 cluster mode 的区别](http://baixin.ink/2018/04/28/spark-mode/)。
Worker node|集群中可以运行应用程序代码的节点。
Executor|在 worker node 上为运行应用程序启动的进程，可以执行任务和存储数据，每一个应用都有其单独的 executor。
Task|发送到 executor 的工作单元。
Job|由 Spark 的 action 操作（如 save、collect）触发包含多个并行 task 的操作集合。
Stage|一个 Job 会根据是否有 shuffle 操作分成多个 stage，每个 stage 会包含相同的在不同的 executor 上并发执行 task。

## 部署流程
Spark 应用运行在 driver 进程中，主要由 `SparkContext` 协调管理，当开始在一个集群中运行时，SparkContext 首先连接到一个集群管理器（如 Spark 自身的 standalone、Mesos 或者 Yarn），集群管理器的作用是为 Spark 应用分配资源。当 SparkContext 连接到集群管理器以后，就可以为应用程序分配集群资源，分配的集群的不同机器就叫 `excutor` 节点，这些节点用来执行 Spark 应用中计算和存储数据的程序。因此，当 SparkContext 连接到这些 executor 以后，就会把应用代码发送过去，最后会把分配好的 task 发送到这些 executor 去执行。

![](http://spark.apache.org/docs/latest/img/cluster-overview.png)

这里注意几点：

- 如果有多个 Spark 应用需要部署在同一个集群内，这些应用的 driver 和 executor 都是相互独立的，各自运行在不同的 JVM 中，同时互相之间也不可以共享资源。
- 集群管理器相对 Spark 应用来说是一个独立封装的部分，即集群管理器如何分配资源是它自己的事情，只要保证最后能给 Spark 应用分配到资源就好了，所以这里我们可以使用不同的集群管理器。
- 由于 driver 和 executor 必须保持通信，所以两者最好处在同一个网段下。

## 如何向集群提交应用
在提交应用之间首先要把应用代码打包，在 Scala 中一般使用 [sbt-assembly](https://github.com/sbt/sbt-assembly) 来打包，可以把所有依赖的包都打在一起，但是注意与 Spark 或者 Hadoop 相关的依赖都要添加 `provided`，保证打包时不会包含这些依赖。打好包以后就可以把 Jar 包上传到集群准备提交应用了，提交命令如下：

```scala
./bin/spark-submit \
  --class <main-class> \
  --master <master-url> \
  --deploy-mode <deploy-mode> \
  --conf <key>=<value> \
  ... # other options
  <application-jar> \
  [application-arguments]
```

下面是用 Yarn cluster 模式提交应用的一个示例：

```scala
spark-submit \
--class ink.baixin.ripple.spark.RippleJob \
--master yarn \
--deploy-mode cluster \
--conf spark.driver.memory=1g \
--conf spark.executor.memory=2g \
--num-executors=2 \
--executor-cores 2 \
./ripple-jobs.jar \
-t streaming
```

## 工作调度
Spark 调度资源的方式主要有两种，前面有讲过，不同的 Spark 应用部署在集群的时候，他们的 executor 进程都是相互独立的，所占用的资源也都是相互独立的，这种调度成为应用之间的调度，由集群管理器控制。而在一个 Spark 应用内部，多个 job 可能会在多个线程上并发执行，针对每一个 SparkContext，Spark 提供了 fair schduler 来调度资源。

#### 应用间调度


## Refer
[Spark 官方文档](http://spark.apache.org/docs/latest/cluster-overview.html)
