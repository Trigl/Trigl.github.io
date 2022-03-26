---
layout:     post
title:      "「Notes」分布式快照算法：Chandy-Lamport 论文阅读"
#subtitle:   "YARN 架构 && Flink on YARN 简介"
date:       2022-01-22
author:     "Ink Bai"
catalog:    true
header-style: "text"
#header-img: "/img/archive-bg.jpg"
tags:
    - 论文学习
    - 分布式系统
    - Flink
---

> 学习 Flink Checkpoint 时，必然要了解分布式系统中实现快照的方法，本文是经典分布式快照 Chandy-Lamport 算法的小记，论文地址：[Distributed Snapshots: Determining Global States of Distributed Systems](https://lamport.azurewebsites.net/pubs/chandy.pdf)

## 解决的问题
Chandy-Lamport 算法的目标是在分布式系统下保存全局的状态，完成全局性的 snapshot，用于系统故障后的恢复，多个节点基于快照恢复到故障前的状态。

单机状态下完成 snapshot 很容易，上锁停止写入新数据然后 snapshot 即可，分布式系统下就比较麻烦了。一种可以想到的思路是通过分布式锁，但也意味着更大的性能开销和数据延迟，并不是一种可接收的方案。Chandy-Lamport 算法就是一种非常直接的、无需加锁的生成快照的方案，下面我们看一下它的原理。

## 原理
#### 数据模型
将分布式系统简化为两部分，这两部分组成一个有向图：

- 有限个进程 process（顶点）
- 进程之间的通信通道 channel（边）

![](/img/content/Xp1jwPx7luS3eoy.png)


那么分布式系统下需要记录的全局状态就是进程的状态和通道中的 message 组成，也就是 snapshot 需要记录的内容。

#### 前置条件

- 必须构成有向图，保证消息可以传递到系统的每个节点
- 节点间网络可靠
- 消息有序，也就是通信需要走 TCP 协议

#### 工作流程
Chandy-Lamport 算法允许从系统中的任意一个进程发起 snapshot，整体流程如下：

1. 发起 snapshot 的进程进行先保存自身状态
2. 生成一个信息 marker，marker 用来标识 snapshot，会和普通 message 走一样的通道
3. 向下游进程传播 marker
4. 保存 marker 生成之后接收到的 input channel 中的 message
5. 下游进程接收到 marker 之后重复执行上面步骤，即先保存自身状态，再下发 marker，再保存接收到的 input channel 的 message
6. 所有的进程都收到了 marker 信息，都完成保存自身状态和 input channel 的状态（也就是保存 message），即认为 snapshot 完成

举个 🌰 ：假如一个系统包含进程 P1 和 P2，P1 进程的状态包含三个变量 X1、Y1 和 Z1，P2 进程包含 X2、Y2 和 Z2。

首先由 P1 发起 snapshot，P1 先记录自身状态，然后生成 marker 信息发送给 P2。而 P2 此时正向 P1 发送普通信息 M1。

![](/img/content/v2-710946564a36e3aedcdb7d16d3d62c14_1440w.jpg)

P2 收到 marker，记录自身状态。P1 收到 P2 发送的 M1，由于 M1 是 P1 生成 marker 之后才传过来的，需要保存这条消息。

![](/img/content/v2-d27ee3d2a979effafe80d67a57e630a2_1440w.jpg)

最后当 marker 再一次传递给 P1 的时候，说明所有进程都已收到 marker，snapshot 结束，snapshot 保存的内容如图中蓝色部分所示。

![](/img/content/v2-817024acebbf660f28102d6ef456a980_1440w.jpg)

## 总结
以上就是 Chandy-Lamport 论文的核心原理，可以看出整体思想其实非常简单直观，最巧妙的一个地方就是通过自定义一个 marker 信息放入整个系统中去传播，将系统中源源不断的消息进行了区分，一部分是 snapshot 前不需要保存的，一部分是 snapshot 后需要保存的，从而保证 snapshot 的数据是完整的。
