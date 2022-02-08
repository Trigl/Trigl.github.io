---
layout:     post
title:      "Flink on YARN 部署详解（I）"
subtitle:   "YARN 架构 && Flink on YARN 简介"
date:       2022-02-08
author:     "Ink Bai"
catalog:    true
header-style: "text"
tags:
    - Flink
---

> Flink on YARN 应该是生产环境中最常见也是最成熟的部署方式了吧，当然 on K8s 也是比较多的，这里我们就探究 Flink 在 YARN 上是如何提交、部署、运行、管控资源的。

## YARN 简介
Apache Hadoop YARN（Yet Another Resource Negotiator，另一种资源协调者），是一个通用的资源管理系统和调度平台，可以为基于其运行的任务、应用提供统一的资源管理和调度。主要有两大功能：

- 资源的统一管理和调度： 集群中所有节点的资源(内存、CPU、磁盘、网络等)抽象为Container。计算框架需要资源进行运算任务时需要向YARN申请 Container， YARN按照特定的策略对资源进行调度进行 Container 的分配。
- 资源隔离： YARN使用了轻量级资源隔离机制Cgroups进行资源隔离以避免相互干扰，一旦Container使用的资源量超过事先定义的上限值，就将其杀死。

#### 架构
YARN 主要架构如下图，总体上是 Master/Slave 架构：

![](/img/content/Nodo.jpg)

YARN 的两个基础组件为：

- ResourceManager(RM)

  YARN 的一个全局的资源管理器，负责对各NM上的资源进行统一管理和调度，主要由两个组件构成，调度器和应用程序管理器。

    1. 调度器(Scheduler)

    调度器根据容量、队列等限制条件（如每个队列分配一定的资源，最多执行一定数量的作业等），将系统中的资源分配给各个正在运行的应用程序。

    调度器仅根据各个应用程序的资源需求进行资源分配，而资源分配单位是 Container，从而限定每个任务使用的资源量。

    Shceduler 不负责监控或者跟踪应用程序的状态，也不负责任务因为各种原因而需要的重启（由 Applications Master 负责）。

    总之，调度器根据应用程序的资源要求，以及集群机器的资源情况，为应用程序分配封装在 Container 中的资源。

    2. 应用程序管理器(Applications Manager)

    应用程序管理器负责管理整个系统中所有应用程序，包括应用程序提交、与调度器协商资源以启动 AM、监控 AM 运行状态并在失败时重新启动等，跟踪分配的 Container 的进度、状态也是其职责。

- NodeManager (NM)

  NM 是每个计算节点上的一个进程，负责每个节点上的资源和任务管理器。

  它会定时地向 RM 汇报本节点上的资源使用情况和各个 Container 的运行状态；同时会接收并处理来自 AM 的 Container 启动/停止等请求。

除了上面两个基本组件，还有两个比较核心的组件是 Container 和 ApplicationMaster。

- Container

  YARN 分配的资源的最小单位是 Container，可以认为 Container 就是内存、CPU、磁盘、网络等资源的集合。

  上面说了每个物理节点下面都有一个 NM，NM 用于管理分配所在节点下的 Container。

- ApplicationMaster (AM)

  用户提交的应用程序必须包含一个AM，负责应用的监控，跟踪应用执行状态，重启失败任务等。

  AM 也是需要运行在 Container 内的，它需要向ResourceManager协调资源，并且与 NM 协同工作完成 Task 的执行和监控。

#### YARN 工作流程
一个应用从客户端提交到 YARN 上的流程如下图：

![](/img/content/yarn-work.jpg)

1. client 向 RM 提交应用
2. RM 为应用程序分配第一个 Container，并与对应的 NM 通信，要求它在这个 Container 中启动 AM
3. AM 启动时向 RM 注册自己，启动成功后与 RM 保持心跳
4. AM 向 RM 申请资源
5. —旦 AM 申请到资源后，便与对应的NM通信，要求它初始化 Container，启动任务
6. NM 启动 Container，运行任务
7. Container 运行期间，Container 通过 RPC 协议向对应的 AM 汇报自己的进度和状态信息
8. 应用运行结束时，AM 向 RM 注销自己，以便 RM 进行资源的回收

## Flink on YARN 架构
Flink JobManager 与 YARN AM 运行在同一个 Container 中，而后续申请的 Container 中则会运行 Flink TaskManager。

![](/img/content/flink-yarn.jpg)

Flink on YARN 也可以细分为 session 模式部署、per-job 模式部署和 application 模式部署，这部分内容详见：[Flink 部署模式](/2021/08/26/flink-architecture/#部署模式)
