---
layout:     post
title:      "「Notes」Flink 轻量级异步快照论文"
subtitle:   "Flink Checkpoint 的原理"
date:       2022-01-23
author:     "Ink Bai"
catalog:    true
header-style: "text"
#header-img: "/img/archive-bg.jpg"
tags:
    - Flink
    - 论文学习
---

> 在 [「Notes」分布式快照算法：Chandy-Lamport 论文阅读](/2022/01/22/chandy-lamport-paper/) 我们介绍了分布式系统下如何实现全局性快照，Flink 基于这篇论文进行了改进，实现了一种更轻量级的异步屏障快照算法（asynchronous barrier snapshotting, ABS），具体论文是 [Lightweight Asynchronous Snapshots for Distributed Dataflows](https://arxiv.org/pdf/1506.08603.pdf)，本文会结合这篇论文的内容讲解 Flink Checkpoint 的原理。

通过本文你可以了解到以下内容：

> **1. Chandy-Lamport 论文实现快照的问题**
> **2. ABS 算法做了哪些优化**
> **3. ABS 算法实现原理**
> **4. 如何解决 barrier 对齐导致的数据延迟问题**

## Chandy-Lamport 算法回顾
首先回顾下 Chandy-Lamport 论文中实现 snapshot 的原理：分布式系统中存在多个进程 process，进程之间通过通道 channel 进行通信，进程和通道构成有向图。snapshot 时会生成一种特殊的标识信息 marker 在有向图中传递，当 marker 传递到所有的进程后，保存各个进程的状态和上游通道中的消息，也就完成了一次 snapshot。

这个过程会保存两部分内容：

- 有向图的顶点，也就是进程自身的状态
- 有向图的边，也就是上游通道中的信息

在流处理系统中，数据是源源不断流入的，通道中的信息量会非常的大，这就会导致需要保存的数据量很大，系统恢复的时间较长。

## Flink ABS 算法的优化
Flink ABS 算法基本沿用了 Chandy-Lamport 的数据模型，整体是一个有向图，在 Flink 中叫 JobGraph，图的顶点称为算子（operator），边是 channel。

- 对于有向无环图（DAG），ABS 算法仅保存顶点也就是算子的状态，不需要保存 channel 中的消息。
- 对于有向有环图（DCG），ABS 算法只需要额外保存少量 channel 中的消息即可。

## ABS 算法实现原理
#### Barrier
Chandy-Lamport 算法中使用 marker 消息作为快照的边界，ABS 算法采用了类似的机制，称为检查点屏障（checkpoint barrier），来分割上一次 checkpoint 和下一次 checkpoint 的数据。

![](/img/content/stream_barriers.svg)

ABS 算法时候能够保证 checkpoint 时仅保存 operator 的状态，不必保存 channel 中记录，主要依赖的是 barrier 对齐（barrier alignment）。

barrier 对齐指的是拥有多个输入流的 operator，当收到某一个输入流的 barrier 时，并不会立刻进行 snapshot，只有所有的输入流的 barrier 都到达后，才开始进行 snapshot，也就是达到多个输入流 barrier 对齐。并且注意，当 operator 收到某个输入流的 barrier，它就会阻塞这个输入流的后续消息，放入到一个 input buffer 内，直到 barrier 对齐之后才取消阻塞，让数据继续向下游流动。这种通过短暂阻塞下一个 checkpoint 数据的方式保证了在 snapshot 时的状态仅包含上一个 checkpoint 的数据，也就不需要额外再保存 input channel 中的数据，大大减少了需要保存的状态的大小，可以看做是一种 trade-off。

![](/img/content/stream_aligning.svg)

#### 有向无环图中的 ABS 实现
论文中通过一个 WordCount 的例子讲解了在有向无环图中 ABS 算法的实现，dataflow graph 如下图：

![](/img/content/195230-47fbcee019af584b.webp)

ABS 算法的执行流程如下：

1. JobManager 的协调器在所有 source 节点插入 barrier
2. source 节点接收到 barrier 时，snapshot 自己的状态（数据源的 offset/position 信息），然后将 barrier 广播给所有的下游节点（上图 a）
3. 非 source 节点收到输入流的 barrier，会阻塞当前输入流，直到接收到所有上游输入流的 barrier，此时才会进行 snapshot，并将 barrier 广播到下游（上图 b、c）
4. 快照生成后，节点不再阻塞输入流，sink 节点接收到 barrier 之后会保存状态并向 JobManager ack，当所有 sink 节点全部 ack，则这次 checkpoint 完成

#### 有向有环图中的 ABS 实现
对于 DCG 图，如果采用 barrier 对齐的方式做 checkpoint，可能会造成死锁，成环的两个算子可能会互相等待对方发送 barrier。

![](/img/content/195230-36d1eef7ff7c1026.webp)

ABS 将形成环的 channel，即从下游流向上游的 channel，称为回边（back edge），由于回边的存在导致我们无法单纯地通过每个算子的状态合并出全局快照，会漏掉从回边流过来的消息。对此的处理方式如下：

- 首先，对于回边终点的那个节点，当该节点其他正常输入流的 barrier 都到达之后，它就先 copy 自身状态（上图 b）
- 从这个时间点开始，该节点会将从回边流入的消息记录下来直到接收到了本次 checkpoint 的 barrier，上一步 copy 的状态和这一步保存的 channel 中的消息都会作为 snapshot 的一部分（上图 c）

#### 异步
ABS 算法的异步体现在保存状态的时候，算子 barrier 进行 snapshot 时，并不会同步等待状态写入完成，而是后台异步写入，主流程会继续开始处理数据流，这样保证了最大可能的降低延迟。

但是异步也带来一个问题，就是各个算子完成 snapshot 的时间无序了，因此 checkpoint 成功的条件应当是所有有状态的算子都要保证向 JobManager ack 自己已经 snapshot 完成。

## 解决 barrier 对齐导致的延迟
上面 ABS 算法中的 barrier 对齐，可以保证 Flink exact-once 语义，存在的一个缺点就是会阻塞一定时间的输入流，导致数据反压，进而延迟。那么有没有办法解决这个问题呢？
#### 仅保证 at-least-once 语义
Flink 可以设置多种语义保证，对于那些无法容忍延迟但是可以容忍重复的任务，我们可以设置成 at-least-once 语义，在这个语义下，不会进行 barrier 对齐。

此时当节点接收到上游数据流的 barrier 后，不会再阻塞数据流，而是一直等到所有上游的 barrier 都到达后，直接开始 snapshot。可以想到，这个过程必然导致我们的状态中包含了 barrier 之后的数据，也就是多了本应属于下一次 checkpoint 的数据，在 Flink 故障恢复的时候就会导致数据重复。

#### Flink 1.11 新特性：非对齐 Checkpoint
对于高反压的场景，反压和对齐 checkpoint 可能会互相影响，比如流量突增导致反压，由于 barrier 到达依赖于数据及时向下游流动，这样就会导致 barrier 对齐时间变长，checkpoint 时间变长。当 checkpoint 时间变长后，算子的 input buffer 很容易就满了，又会反过来加剧反压。这样极端情况可能导致数据延迟非常久。

因此 Flink 1.11 实现了新的非对齐 Checkpoint 来解决这个问题，基本原理还是源于 Chandy-Lamport 算法，就是各个节点不再对其多个上游输入进行 barrier 对齐，只要接收到上游 barrier 之后就开始本身的状态，然后立即向下游广播 barrier，不会阻塞数据流。当有其他上游消息进入的时候，由于其他上游 barrier 还没到，就需要保存 barrier 到达之前的这些消息作为 snapshot 的一部分，保证通过 Checkpoint 恢复的时候能够拿到所有的数据。整体来看非对齐 Chekcpoint 就是用空间换时间。

显而易见，这样带来的问题就是持久化的数据变多，对磁盘的压力增大，同时作业的恢复时间也会变长。因此没有哪种方案能够完美解决所有问题，非对齐 Checkpoint 可能更适合那些容易产生高反压的复杂作业，对于只是做 ETL 的简单作业而言，更轻量级的对齐 Checkpoint 显然是更优选。

## 总结
本文详细介绍了 Flink Checkpoint 的实现基础，ABS 算法，包含的内容如下：

- ABS 实现原理
  - barrier 对齐：节点需要等待所有上游节点 barrier 都到达之后再开始进行 snapshot
  - 异步：后台异步写入状态，不阻塞数据处理主流程
  - Checkpoint 流程：JobManager 协调器定期触发 Checkpoint，向所有 source 算子发送 barrier，barrier 一路向下传播到 sink 节点，非 source 算子节点通过 barrier 对齐机制做 snapshot，snapshot 仅保存自身状态即可，不需要保存 channel 中的数据，当所有算子的所有节点 snapshot 都成功之后，向 JobManager ack，checkpoint 完成。
- 解决 barrier 对齐导致的延迟问题
  - barrier 对齐是为了实现 exactly-once，只要 Flink 设置为 at-least-once 就不会再进行 barrier 对齐，带来的问题就是数据重复。
  - Flink 1.11 实现了非对齐 Checkpoint，完整实现了 Chandy-Lamport 算法，通过空间换时间的方式使 Checkpoint 不再阻塞数据流，非常适合高反压作业。

## Refer
[Paper 阅读: Lightweight Asynchronous Snapshots for Distributed Dataflow](https://matt33.com/2019/10/20/paper-flink-snapshot/)
[深入理解Flink的轻量级异步屏障快照（ABS）算法](https://www.jianshu.com/p/3093f6d92750)
[Flink 1.11 新特性详解:【非对齐】Unaligned Checkpoint 优化高反压](https://mp.weixin.qq.com/s/rxxpePoh-Z2fRwQexwsbxg)
