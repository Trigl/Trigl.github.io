---
layout:     post
title:      "Flink 集群的组成"
date:       2021-08-26
author:     "Ink Bai"
catalog:    true
header-style: "text"
tags:
    - Flink
---

> 本文介绍了 Flink 架构，描述了其主要组件如何交互以执行应用程序和从故障中恢复。

Flink 采用的是分布式系统中标准的 master-slave 架构，运行时包含两个主要组件：一个 JobManager 和一个或多个 TaskManager。当 Flink 集群启动后，首先会启动一个 JobManager 和一个或多个 TaskManager。由 Client 提交任务给 JobManager，JobManager 再调度任务到各个 TaskManager 去执行，然后 TaskManager 将心跳和统计信息汇报给 JobManager。TaskManager 之间以流的形式进行数据的传输。上述三者均为独立的 JVM 进程。

![](/img/content/flink-arc.jpg)

## Flink Client
Client 不是运行时和程序执行的一部分，client 端会将流或者批任务编译成一个数据流图，然后将其提交到 JobManager。

## JobManager
JobManager 的主要用于调度 Flink 应用的分布式执行，具体职责有：调度 task、对完成的 task 或者失败的 task 做出反应、协调全局 checkpoint、全局容错恢复等。这个进程内又由三个不同组件组成：

- ResourceManager：我们不创造资源，我们只是资源的搬运工。ResourceManager 负责 Flink 集群中的资源分配、回收，它管理的是 task slots，task slot 是 Flink 集群中资源的最小单位。Flink 仅仅是一个 manager，它为不同的环境和资源提供者（如 YARN、Mesos、Kubernetes 和 standalone 部署）实现了对应的 ResourceManager。
- JobMaster：负责管理单个 Flink Job 的执行，Flink 集群中可以同时运行多个作业，每个作业都有自己的 JobMaster
- Dispatcher：提供了一个 REST 接口，用来提交 Flink 应用程序执行，并为每个提交的作业启动一个新的 JobMaster，同时它还运行 Flink WebUI 用来提供作业执行情况

## TaskManager
也称为 worker，用来实际执行 task，并且缓存和交换数据流。每个 worker 都是一个 JVM 进程。

#### task 和 subtask
这里要理解 task 和 subtask 的概念，Flink 一个 job 是由多个 task 组成的，task 可以理解成是一个逻辑概念，而它的物理实体就是 subtask。比如一个 task 是 map 操作，我们设置这个操作的并行度是 5，实际执行的时候也就会启动 5 个线程去做这个 map 操作，这 5 个线程就是 5 个 subtask，即一个线程真正去执行的是 subtak。

![](/img/content/flink-tasks.png)

这里还有一个概念叫 operator chain，即算子链，如果两个相邻算子之间没有发生数据 shuffle，那么它们就可以合而为一形成一个整体叫做算子链，这个算子链也是一个 task。

上面提到了数据 shuffle，shuffle 也就是洗牌，我们知道 Flink 是并行执行的，一个并行下的数据流称为一个分区，比如算子 A 有 5 个并行度，那么算子 A 处理数据时也就分成了 5 个分区，如果传输到算子 B 的时候，仍然是 5 个分区，并且原来属于算子 A 下某个分区的数据到了算子 B 还在对应的分区内，我们就说数据没有发生 shuffle，比如常见的 map 算子、filter 算子。但是像 keyBy 这种算子，同一个 key 的元素需要聚合在一起，那么就会发生原本属于同一分区下的数据按照 key 分配到了不同的分区，这就叫数据发生了 shuffle。

发生 shuffle 就意味着数据需要通过网络在 task 之间进行传递，会涉及到序列化、磁盘读写、socket 读写与反序列化等多个过程。而相同数据在内存中通过方法调用的方式传输，仅需要传输对象指针即可，消耗非常小。

目前 Flink 提供了 chaining 算子的机制将相邻算子链接到一个 task 内来避免 shuffle，它能减少线程之间的切换，减少消息的序列化与反序列化，减少数据在缓冲区的交换，减少数据在 TM 之间的网络传输，并减少延迟的同时提高整体吞吐量。但是 chaining 算子存在诸多限制，其中一条就是“下游算子的入度为 1”，也就是说下游算子只能有一路输入。这就将多路输入的算子（如 join）排除在外。

#### slot
讲完 task 和 subtask 再来说说 task slot，TaskManager 中资源调度的最小单位是 task slot，一般一个 TaskManager 中可以有多个 slot。例如，具有 3 个 slot 的 TaskManager，会将其托管内存 1/3 用于每个 slot。分配资源意味着 subtask 不会与其他作业的 subtask 竞争托管内存，但是注意此处没有 CPU 隔离，当前 slot 仅分离 task 的托管内存。

上面说了 task 实际上是分配到 subtask 执行的，那么 subtask 如何使用 slot 的资源呢？

默认情况下，Flink 允许 subtask 共享 slot，即便它们是不同 task 的 subtask，只要是来自于同一作业即可。允许 slot 共享有两个主要优点：
- Flink 集群所需的 task slot 和作业中使用的最大并行度恰好一样。一般一个作业有多个 task，这些 task 各自设置的并行度可能不一样，当我们允许 subtask 共享 slot 的时候，我们就可以看哪个 task 设置的并行度最大，然后直接把作业的并行度设置为这个最大值即可。
- 容易获得更好的资源利用。由于各个 slot 的资源都是一样的，如果没有 slot 共享，非密集 subtask（source/map）将占用和密集型 subtask（window） 一样多的资源，这就产生了资源浪费。

如这张图，我们不设置 subtask 共享 slot，每个 slot 下面只跑一个 subtask，各个 slot 的资源利用率不均匀。
![](/img/content/no-share-slot.png)

相反，如下图，subtask 可以共享 slot 时，基本并行度从上面中的 2 增加到 6，可以充分利用分配的资源，同时确保繁重的 subtask 在 TaskManager 之间公平分配。
![](/img/content/share-slot.png)

## 部署模式
首先明确下面两个术语：

- Flink Application：Flink Application 是一个 Java 应用程序，它通过 main 方法(或其他方式)提交一个或多个 Flink Job。提交作业通常通过调用 execute 来完成。
- Flink Job：是 Flink DAG 图在运行时的具体表现，一个 DAG 对应一个 Flink Job，一般是通过在 Flink Application 中调用 execute 来创建和提交的。

然后再来看 Flink 如何区分不同的部署模式：

- 集群的生命周期和隔离性保证
- application 的 main 方法在哪里执行

![](/img/content/deployment_modes.png)

- Application 模式
隔离性：仅执行同一个 Application 的 Job（一个或多个），生命周期和 Application 的生命周期相同。
application main 执行位置：Job Manager，相对比在 client 端执行，可以节省 CPU 和下载任务依赖的带宽消耗。

- Per-Job 模式
隔离性：仅执行一个 Job，生命周期和这个 Job 相同。
application main 执行位置：client

- Session 模式
隔离性：可以执行多个 Application 的 Job，共享 JobManager，某一个 Job 结束并不会结束其生命周期。
application main 执行位置：client





## Refer
[Flink 架构](https://ci.apache.org/projects/flink/flink-docs-release-1.13/zh/docs/concepts/flink-architecture/)
[Flink 部署](https://nightlies.apache.org/flink/flink-docs-master/zh/docs/deployment/overview/)
