---
layout:     post
title:      "Flink 学习大纲"
date:       2021-08-25
author:     "Ink Bai"
header-style: "text"
catalog:    true
tags:
    - Flink
---

> 随着阿里的强势入主，Flink 目前已经成为了国内事实上的标准流式处理框架，本文将对其做一个基本的介绍，作为后续深入学习的大纲。
本文主要回答以下三个问题：
- Flink 到底是什么？解决了什么问题？
- Flink 包含哪些基本内容？
- Flink 学习资源推荐

## Flink 是什么？
官方文档里说，Flink 是一个框架和分布式处理引擎，用于在无边界和有边界数据流上进行有状态的计算。

首先 Flink 是一个框架，也就意味着已经为我们包装好了各种功能和组件，我们直接可以拿来方便的使用。
那么用来干嘛呢，它的作用就是一个分布式处理引擎，处理的是数据流。
数据流又分位无边界和有边界的数据，无边界也就是流式数据，有边界也就是批数据。
最后，强调了进行的是有状态计算，一般任务分位有状态计算和无状态计算。
无状态计算指的是处理一条数据的结果仅跟这条数据本身有关，输入不变的情况下输出一定也不变。
而有状态计算指的是处理一条数据，除了要计算该条数据本身，还要与其他有状态的数据结合起来，比如说一些需要时间窗口的计算。

## 要掌握 Flink 需要学习哪些内容
作为一个成熟强大的框架，Flink 包含的内容非常多，我认为需要掌握的核心内容有：
- 流计算常见概念和基本知识
- Flink 的整体架构，包括分层 API、Flink 集群的部署、Flink 任务的执行
- Flink 的状态管理，包括 State 的底层实现、State 的类型、基于 State 实现容错等等
- Flink 关于时间的内容，watermark 和 window 等
- Flink DataStream API
- Flink 的 connectors
- Flink 网络栈，如何通信
- Flink Table API/SQL 相关内容
- Flink Debug 和监控相关的内容
- Flink 的优缺点，与其他流处理引擎的对比
- Flink 的业界实践

## 学习 Flink 的一些资源
[Flink 中文学习社区](https://flink-learning.org.cn/activity/detail/bdbef5a3caf3029595b8c64d556c9811)：有一些入门和进阶的视频。
[Flink 知识图谱](/files/Apache-Flink-Stateful-Computations-over-Data-Streams.pdf)
[Streaming System](https://book.douban.com/subject/27080632/)：讲流处理引擎的一本书，书本的质量非常高，配了大量的图，目的就是让你很容易的懂流处理引擎中的概念（比如时间、窗口、水印等）。
[Lightweight Asynchronous Snapshots for Distributed Dataflows](https://arxiv.org/pdf/1506.08603.pdf)：Apache Flink 所实现的一个轻量级的、异步做状态快照的方法。基于此，Flink 得以保证分布式状态的一致性，从而保证整个系统的 exactly-once 语义。
[Distributed Snapshots: Determining Global States of Distributed Systems](https://lamport.azurewebsites.net/pubs/chandy.pdf)：分布式快照算法 Chandy-Lamport
[MillWheel: Fault-Tolerant Stream Processing at Internet Scale](https://static.googleusercontent.com/media/research.google.com/zh-CN//pubs/archive/41378.pdf)：MillWheel 是 Google 内部研发的实时流数据处理系统，具有分布式、低延迟、高可用、支持 exactly-once 语义的特点。
[Streaming 101: The world beyond batch](https://www.oreilly.com/radar/the-world-beyond-batch-streaming-101/)
[Streaming 102: The world beyond batch](https://www.oreilly.com/radar/the-world-beyond-batch-streaming-102/)
