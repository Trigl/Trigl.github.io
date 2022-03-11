---
layout:     post
title:      "ZooKeeper 故障时如何保证 Flink JobManager 的高可用？"
subtitle:   "Flink on YARN 模式下 HDFS + ZooKeeper 实现 HA"
date:       2022-03-11
author:     "Ink Bai"
catalog:    true
#header-style: "text"
header-img: "/img/archive-bg.jpg"
tags:
    - Flink
---

> 上篇文章 [Flink JobManager 高可用详解](/2022/02/11/flink-ha/) 介绍了 Flink JobManager 的高可用及其实现，文章后面讨论了一种异常场景：
>**「若 ZooKeeper 发生了比较大的故障，宕机时间比较久，此时如何保障 Flink JobManager 的高可用？」**
> 带着这个问题，今天进行一些更深入的探索。

## 移除 ZooKeeper？
在 Flink On YARN 模式下，仅会存在一个 JobManager，不会有其他备选 JobManager 节点，那么是否可以考虑使用一个更加稳定的分布式存储如 HDFS 来替换 ZooKeeper 呢？我们分析以下可行性：

- leader 选举：仅有一个 JobManager，不需要进行选举，leader 地址信息写入 HDFS 即可，写入频率很低，可行
- leader 发现：HDFS 没有类似于 ZooKeeper 的主动通知机制，需要 HDFS Client 主动轮询 HDFS 路径，读取比较频繁。若为了快速发现 leader 地址发生变化，就需要缩小轮询间隔，也就意味着对 HDFS 压力增大，可行但是不完美
- 持久化数据：全部存储到 HDFS 即可，可行

通过上面分析，若要使用 HDFS 完全替换 ZooKeeper，存在的最大问题是如何平衡 leader 发现的时效性和持续轮询对 HDFS 造成的压力。

## HDFS + ZooKeeper 实现 HA
既然单独使用 ZooKeeper 或者 HDFS 实现 HA 都不是很完美，那么我们可以考虑将这两者结合起来，设计如下：

![](/img/content/flink-ha-write.jpg)

leader 的地址信息同时写入 HDFS 和 ZooKeeper，leader 发现时优先读取 ZooKeeper，HDFS 作为 fallback。

#### 如何保证 HDFS 和 ZooKeeper 的数据一致性？
由于需要同时往 HDFS 和 ZooKeeper 写入相同数据，因此可能存在数据不一致的问题，解决不一致问题的常用手段就是加版本号信息，我们给写入 HDFS 和 ZooKeeper 的数据都加上版本号，只要两者的版本号一致就可以认为数据一致。

在 [Flink JobManager 高可用详解](/2022/02/11/flink-ha/) 讲过 leader 选举会存储类 `LeaderInformation` 中的成员变量，每一个新的 JobManager 都会生成一个新的 sessionId，它是一个 UUID 随机字符串，我们可以把这个类作为版本号。

```Java
/** Information about leader including the confirmed leader session id and leader address. */
public class LeaderInformation implements Serializable {

	private static final long serialVersionUID = 1L;

	@Nullable private final UUID leaderSessionID;

	@Nullable private final String leaderAddress;

	private static final LeaderInformation EMPTY = new LeaderInformation(null, null);

	private LeaderInformation(@Nullable UUID leaderSessionID, @Nullable String leaderAddress) {
		this.leaderSessionID = leaderSessionID;
		this.leaderAddress = leaderAddress;
	}
}
```

#### leader 选举过程
![](/img/content/leader-se.jpg)

- 必须保证写入 HDFS 成功
- 如果 leader 信息写入 HDFS 失败时直接停止任务，是因为 Flink 任务强依赖 HDFS，如做 checkpoint 持久化，如果 HDFS 有异常那么启动起来也没有意义。
- 只要成功写入 HDFS 就可以正常启动任务了，即使 ZooKeeper 写入失败也不影响任务正常启动。
- ZooKeeper 写入失败后放到异步线程不停重试，直到写入成功，避免阻塞启动流程，同时保证 ZooKeeper 和 HDFS 数据一致。
- 正常写入流程只发生在 JobManager 启动时，写成功就结束了。我们之前讲过 ZooKeeper 写的是临时节点，若由于偶然网络不稳定导致 session 断连临时节点被删除，那么 ZooKeeper 数据就和 HDFS 数据不一致了，因此我们需要有一个基于 ZooKeeper Watcher 进行数据回写的逻辑。

#### leader 发现过程
leader 发现有两种模式，ZooKeeper 正常时基于 发现有两种模式，ZooKeeper 进行 leader 发现有两种模式，ZooKeeper 异常时基于 HDFS 进行 leader 发现，而这两种模式通过 zkAvailable 这个字段进行切换。

**1. ZooKeeper leader 发现**

![](/img/content/leader-zk.jpg)

- 使用 ZooKeeper Watcher 机制监听节点变化、异常信息，来进行 leader 发现和 ZK 模式切换。
- ZooKeeper leader 发现的逻辑在主线程内，即其生命周期和任务的生命周期一致，不会被提前销毁。

**2. HDFS leader 发现**

![](/img/content/leader-hdfs.jpg)

- 服务发现节点需要主动 poll leader 信息，检测是否更新
- ZooKeeper 可用之后，要及时结束 HDFS 轮询线程，减少对 HDFS 的压力
- 轮询的时间间隔不能太长也不能太短，太长会超过 JM 到 TM 的心跳时间，导致 TM 被销毁；太短对 HDFS 压力比较大。

## 异常场景分析
分析一下各种异常 case 情况下 Flink 的可用性，主要分析以下异常：

- ZooKeeper 故障
- HDFS 故障
- JobManager GC 异常

#### 任务启动时 ZK 由可用 → 不可用
- **JobManager**：此时 ZK 不可用会影响 leader 选举，数据写 HDFS 成功，写 ZK 失败，但是 JobManager 能够正常启动。
- **TaskManager**：TaskManager 采用 HDFS 进行 leader 发现，读写数据一致，都是 HDFS，TaskManager 正常运行。

#### 任务运行期间 ZK 由可用 → 不可用
- **JobManager**：任务正常运行期间 JobManager 不会重启，也就不会进行 leader 选举，JobManager 无影响。
- **TaskManager**：TaskManager 由 ZK leader 发现转为 HDFS leader 发现，读写数据一致，都是 HDFS，TaskManager 正常运行。

#### 任务运行期间 ZK 由不可用 → 可用
- **JobManager**：数据会重试写入 ZK 内，JobManager 任务状态不变。
- **TaskManager**：TaskManager 由 HDFS leader 发现转为 ZK leader 发现，读写数据一致，都是 ZK，TaskManager 正常运行。

#### JobManager 发生了异常重启但 ZK 仍然不可用
- **JobManager**：leader 选举后数据写 HDFS 成功，写 ZK 失败，JobManager 正常启动。
- **TaskManager**：注意此时由于是 JobManager 异常重启，注意两个点：
  - leader 信息发生了变化
  - TaskManager 之前一直在运行，并且基于 HDFS 轮询进行 leader 发现

如果 HDFS 轮询时间大于 JM 到 TM 的心跳超时，就会导致 TM 被销毁，因此这里设置合适的轮询时间就可以保证 TaskManager 正常运行。

#### 任务运行期间，ZK 正常，HDFS 挂掉
- **JobManager**：任务正常运行且 ZK 正常时，leader 状态不变，但是由于写 HDFS 异常，做 checkpoint 时会失败，其他正常。
- **TaskManager**：ZK 正常，leader 发现基于 ZK 实现，也正常运行。

#### ZK 网络抖动，临时节点可能被删除
- **JobManager**：临时节点如果被删除，leader 选举被 ZK 通知节点变更后，会将临时节点回写到 ZK，保证与 HDFS 数据最终一致。
- **TaskManager**：leader 发现在 ZK 节点删除期间会转换到 HDFS leader 发现，ZK 节点回写后继续转换回 ZK leader 发现。

#### JobManager 由于 GC 长时间卡住的 case
JobManager 进程整体卡住，JobManager 与 YARN 心跳超时，但 YARN 没有启动新的 JobManager。TaskManager 与 JobManager 心跳超时，进程停止。

这种 case 没有太好的解决方法，而且由于 GC 可能导致整个 JobManager 卡住，Flink 任务还不会快速失败，可能还要配置 GC 告警及时发现问题，之后再 case by case 排查 GC 时间长的原因。

## 总结
本文介绍了在 Flink ON YARN 模式下一种新的可以同时基于 HDFS 和 ZooKeeper 来实现 Flink JobManager 高可用的方案，由强依赖 ZooKeeper 变为弱依赖 ZooKeeper，最后分析了各种异常 case 下这种方案的处理，较之强依赖 ZooKeeper 实现 HA 的方案，可用性有了更大的提升。
