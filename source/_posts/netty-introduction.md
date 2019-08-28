---
layout:     post
title:      "高性能网络应用框架 Netty 介绍"
date:       2019-08-29
author:     "Ink Bai"
header-img: "/img/post/netty.jpg"
tags:
    - Netty
---

> Netty 是一个高性能网络应用框架，应用很普遍，在 Java 领域 Netty 基本上是网络编程的标配了，所以很有必要深入学习一下。

## 网络编程的瓶颈在哪？
网络编程都是基于 socker 编程，Java 最基本的网络编程模型是 BIO，即阻塞式 I/O，BIO 中所有的读写操作都会阻塞当前线程。

![](/img/content/bio-thread.png)

如果客户端和服务端建立了一个连接，但是客户端一直没有请求过来，那么服务端的 read() 就会一直处于阻塞状态。如果服务端处理这个请求时在等待其他资源时阻塞了，那么此时客户端就会一直处于阻塞状态，无法继续发送请求。所以使用 BIO 模型时一般都会为每个 socket 分配一个独立的线程。

![](/img/content/multi-bio-thread.png)

为了避免频繁创建、消耗线程，我们可以采用线程池创建线程，但是 socket 和线程的关系的对应关系是不变的。

BIO 这样的线程模型适用于 socket 连接不是很多的场景，但是现在的互联网场景，往往需要服务器支撑成百上千万的连接，而创建上百万个线程显然不现实，BIO 无法满足，我们需要的线程模型应该是这样的：

![](/img/content/nio-thread.png)

这个时候 Java 的 NIO 就粉墨登场了，我们可以利用 NIO 实现一个线程处理多个连接，具体如何实现呢？现在普遍都使用 `Reactor` 模式，Netty 的实现也是基于 `Reactor`，接下来我们先了解一下 `Reactor` 模式。

## Reactor 模式
