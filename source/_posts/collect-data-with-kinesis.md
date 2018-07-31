---
layout:     post
title:      "使用 AWS Kinesis 收集流数据"
date:       2018-05-16
author:     "Ink Bai"
catalog:    true
header-img: "http://ox2ru2icv.bkt.clouddn.com/image/post/aws-kinesis.jpg"
tags:
    - AWS
---

> Kinesis 是 AWS 的一项用于收集实时流数据的云服务，类似于 Kafka。Kinesis 收集到的数据可以用于多个方面，例如存到 S3，发到 EMR 作进一步数据分析等等。

Kinesis 的整体架构如下：

![](https://docs.aws.amazon.com/streams/latest/dev/images/architecture.png)

## Kinesis 的基本术语

**Kinesis Data Stream**
Kinesis Data Stream 实时吸收大量数据、持久存储数据并使这些数据可供使用。其由多个分片组成，每个分片里面包含一系列数据记录，每个数据记录有一个由 Kinesis Data Stream 分配的序列号。

**数据记录**
数据记录是存储在 Kinesis data stream 中的数据单位。数据记录由序列号、分区键和数据 Blob 组成，数据 Blob 可以是 最多 1 MB，并且是不可变的。

**保留周期**
数据记录在添加到流中后保存的时长，默认保存 24 小时，可以设置为更久的值，最长保存 168 小时，但是设置超过 24 小时的保留周期是会额外收费哦。

**分片**
分片可以理解为 Kinesis 流的分区，一个 Kinesis Data Stream 由一个或多个分片组成。每个分片可以支持最大每秒 1000 条记录或 1MB 数据量的写入速度，每秒 5 次事务或者 2MB 数据量的读取速度，Kinesis 流总的数据容量是各个分片之和，所以如果数据速率发生改变，可以相应增减分配给流的分片数量。

**分区键**
每个数据放入流中时，都必须指定一个不超过 256 个字节的 Unicode 字符串作为分区键，Kinesis 将会对这个分区键进行 MD5 哈希映射成一个 128 为的整数值，然后分配到对应的分片中去，这样就实现了对传入的数据分区。
通常，分区键的数量应比分片的数量多得多。这是因为分区键用来确定如何将数据记录映射到特定分片。如果有足够的分区键，数据可以在流中的分片间均匀分布。

**序列号**
每条数据记录均有一个序列号，此序列号在其分片中是唯一的。当向 Kinesis 流中写入记录时，Kinesis Data Streams 将分配序列号。同一分区键的序列号通常会随时间变化增加；写入请求之间的时间段越长，序列号则越大。

**Producer**
Producer 用于产生数据记录并放入 Kinesis Data Stream 中，例如发送日志数据到流的 Web 服务器是 Producer。

**Consumer**
Consumer 从 Kinesis Data Stream 获取记录并进行处理，一个 Kinesis 流可以有多个 Consumer，每个 Consumer 可以同时单独使用流中的数据。

**Kinesis Producer Library (KPL)**
Kinesis Producer Library 是用于编写 Producer 应用的库，包含了批上传 record，容错，监控等功能，可以帮助我们简化 producer 的开发。

**Kinesis Client Library (KCL)**
Kinesis Client Library 是一个用于编写 Consumer 应用的库，简化读取流的操作并且具有容错性。

## 基于 Akka 和 KPL 开发 Producer
流处理的第一步就是数据收集，假设现在有这样的一个需求，访客在页面访问或者点击时，前端会相应向服务器发送一条 Get 请求记录访客的访问或者点击事件，现在需要一个应用把这些 Get 请求內的数据打入到 AWS Kinesis 內以供后面的实时计算用，如何做呢？

其实这个需求简单点说就是实现一个 Producer 把前端 Get 请求的数据发到 Kinesis 中，那么我们需要考虑哪些方面呢？

1. 首先我们必须实现异步传输数据到 Kinesis，因为前端发送过来的是一个 Get 请求，我们需要立刻返回给前端一个结果，所以需要异步处理传输数据的过程。
2. 然后需要考虑的是多线程问题，因为用户量如果很大的话，每秒发过来的请求量也会很大，所以这个 Producer 首先是能够支撑一定的并发量的。
3. 其次就是批量向 Kinesis 传输记录的问题，AWS 向 Kinesis 的 API 具有 putRecords 方法，但这个方法不是原子性的，就是可能会存在某些 record 发送成功但某些 record 发送失败的情况
4. 容错性问题。可能会由于网络等原因导致数据传输失败，这个时候就需要加上一些重试机制。
5. 上传数据带宽吞吐量的问题。在带宽有限的情况下我们肯定是想要请求上传的次数越少，并且每次传输的数据量越大越好。
6. 上传数据大小的问题，Kinesis 每个分片支持最大每秒 1000 条记录或 1MB 数据量的写入速度，所以如果我们压缩数据那就可以实现每秒传输更多的数据。
7. 如何优雅地停止 producer 呢？因为停止一个 producer 的时候可能还有些记录正在上传，必须保证所有记录已经传输成功并且不会有新的数据再进来时再停止 producer。

基于以上多方面的考虑，我们决定构建一个基于 Akka-Actor 的系统，Actor 模型是基于事件可以用来处理多并发的系统，天然支持异步处理，并且具有很好的容错性，关于 Akka 的内容可以查看我的这篇文章：<a href="http://baixin.ink/2018/05/23/akka-study/" target="_blank">Akka Study</a>

而上传记录我们使用的是 AWS 的 KPL，当然也可以使用 AWS 的上传数据的 API，但是 KPL 已经解决并且封装好了我们上面提到的关于批量上传、容错重试和吞吐量的一些问题，例如当数据传过来以后 KPL 并不是立刻上传，而是放在一个 buffer 里面，等到达到了一定量才会批量上传这些记录，这就减少了 Http 请求的次数。

项目代码见我的 github：[kinesis-producer](https://github.com/Trigl/kinesis-producer)

## Refer
[Amazon Kinesis Data Streams](https://docs.aws.amazon.com/zh_cn/streams/latest/dev/introduction.html)
[Implementing Efficient and Reliable Producers with the Amazon Kinesis Producer Library](https://aws.amazon.com/blogs/big-data/implementing-efficient-and-reliable-producers-with-the-amazon-kinesis-producer-library/)
[reactive-kinesis](https://github.com/WW-Digital/reactive-kinesis)
