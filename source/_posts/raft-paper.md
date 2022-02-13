---
layout:     post
title:      "「Notes」Raft 算法论文：In Search of an Understandable Consensus Algorithm"
# subtitle:   "个人笔记"
date:       2022-02-13
author:     "Ink Bai"
catalog:    true
# header-style: "text"
header-img: "/img/archive-bg.jpg"
tags:
    - 论文学习
    - 分布式系统
---

> 本文是 Raft 算法论文的阅读笔记，主要是个人的一些理解，不保证完全正确。
论文地址：[Raft Paper](https://raft.github.io/raft.pdf)

## Raft 用法用来解决什么问题？
Raft 是一种共识算法，它是从复制状态机的背景下提出的，关于复制状态机可以看下这篇文章：[什么是日志复制状态机?](https://www.ideawu.net/blog/archives/1196.html)

简单来说，要实现分布式系统的存储，首要的条件就是实现多副本进行数据冗余，防止单点故障带来的数据丢失问题。但多节点备份又会引起新的问题，如数据不一致、乱序等。

Raft 就是解决分布式系统数据复制问题，允许一组机器像一个整体一样一致地工作，即使其中一些机器出现故障也能够继续工作下去。

## 算法主要内容
Raft 论文作者提到，创造 Raft 算法的目的除了解决一致性问题，更重要的是要 `Understandable`，易于理解。我学习 Raft 的过程中也感觉其设计特别贴合真实世界，与我们现实生活中的投票选举非常相似。

## 容错

## 总结

## Refer
[Raft 官网](https://raft.github.io/)
[什么是日志复制状态机?](https://www.ideawu.net/blog/archives/1196.html)
[中文翻译](https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md)
