---
layout:     post
title:      "聊一聊 Flink 增量 checkpoint 的问题"
#subtitle:   "实现 Full Checkpoint"
date:       2022-02-12 01:00:00
author:     "Ink Bai"
catalog:    true
header-style: "text"
#header-img: "/img/archive-bg.jpg"
tags:
    - Flink
---

本文主要讲解如下内容：

> **1. 增量 checkpoint 原理简介**
> **2. 增量 checkpoint 的问题以及如何解决**

## 背景
#### 增量 checkpoint 简介
首先，简单介绍下什么是增量 checkpoint。

容错机制是 Flink 一个强大的特性，保证了 exactly-once 的语义，而容错的核心机制就是 checkpoint。checkpoint 是一次全局性的、异步的生成应用状态快照的过程，会周期性的触发，最终把快照写入到可靠存储中，通常是分布式文件系统。

早期 Flink 版本中，每次 checkpoint 都会将应用状态全量做一次快照然后上传，对于状态比较大的作业，可想而知，这是一个非常消耗资源并且缓慢的操作。因此在 Flink 1.3 中引入了「增量快照」（incremental checkpoint），它的基本原理是在两次 checkpoint 之间，应用的状态变化不是很大，没有必要对全量状态制作快照，而是只对与上次 checkpoint 不同（增量）的部分制作快照。当前 Flink 仅支持在 RocksDB 状态后端中使用增量 checkpoint，这是因为 RocksDB 支持了增量的写入文件、后台自动合并文件等功能。

以上就是增量 checkpoint 的一个简单介绍，想要了解更多可以看下这两篇文章：

[Apache Flink 大状态管理：增量 checkpointing 介绍](https://www.jianshu.com/p/9540c0f08c44)
[浅谈Flink基于RocksDB的增量检查点机制](https://blog.csdn.net/nazeniwaresakini/article/details/104220180)

#### 增量 checkpoint 存在的问题
尽管增量 checkpoint 在大状态场景下极大减少了 checkpoint 的制作时间，但背后存在一些权衡，也就带来一些问题：

- 每一次 checkpoint 仅生成增量文件，完整状态文件依赖多个 checkpoint
- 由于要从多个 checkpoint 中读取恢复数据，任务恢复时间变久
- 尽管 checkpoint 自身有清理机制，但由于 checkpoint 之间存在依赖关系，旧的 checkpoint 可能并不会被删除，文件数会膨胀

例如我们有一个大状态作业，整个 checkpoint 文件大小超过 10T，shared 共享文件数多达 20W 个，单个算子的文件大小超过 1T

![](/img/content/state-size.jpg)

## 实现类似于 savepoint 的全量 checkpoint
增量 checkpoint 的问题就是为了解决之前全量 checkpoint 的问题带来的副作用，有没有一种办法既享受增量 checkpoint 带来的速度优化，又享受全量 checkpoint 文件数据上的优化呢？理论上增量和全量是矛盾的，没办法综合两者的优势又不带来副作用，但是我们可以对 checkpoint 的状态进行管理来间接实现这一目标：

- 大状态任务正常情况进行增量 checkpoint
- 定期检测 checkpoint 文件大小，超过阈值主动触发一次全量 checkpoint，从而重建 checkpoint 文件依赖关系，减少文件数

Flink 提供了 savepoint 可以对状态进行全量快照，但是 savepoint 是一个工具性的功能，一般用法是任务下线时进行一次 savepoint 创建全量快照，然后再启动时主动选择从 savepoint 恢复。但在 Flink 运行时 checkpoint 的生命周期中，savepoint 产生的全量快照是无法被后续的 checkpoint 依赖的，因此我们需要实现类似于 savepoint 的运行时全量 checkpoint 能力，产生的全量快照可以被后续正常 checkpoint 所依赖，从而使后续 checkpoint 不再依赖于更早之前 checkpoint 的文件。

基于这种全量 checkpoint，我们就有了更多手段去管理任务状态：

- 可以提供 Restful API 去主动触发一次全量 checkpoint
- 可以基于状态数据量和文件数去自动触发全量 checkpoint
- 可以配置固定时间触发全量 checkpoint，如凌晨业务低峰期

## 总结
增量 checkpoint 的原理：checkpoint 之间状态发生的变化不大，没有必要每次全量生成状态文件并且上传远端，因此基于 RocksDB 的特性实现了仅对变化的状态进行增量生成文件的能力，使每次增加的文件数据量大大减少。

增量 checkpoint 的问题：状态文件依赖于多个 checkpoint 的状态文件，文件数和数据量会逐渐膨胀，任务恢复时间变久。

如何解决：主动管理任务状态，在 Flink 正常运行期间可以触发类似于 savepoint 的全量 checkpoint，并且在任务不重启的情况下，这个全量 checkpoint 产生的快照文件可以被后续的 checkpoint 依赖，从而切断先前的 checkpoint 文件依赖链。
