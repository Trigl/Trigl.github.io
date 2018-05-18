---
layout:     post
title:      "使用 AWS Kinesis 收集流数据"
subtitle:   "AWS Kinesis Producer"
date:       2018-05-16
author:     "Ink Bai"
catalog:    true
header-img: "http://ox2ru2icv.bkt.clouddn.com/image/post/winter-akka.jpg"
tags:
    - AWS
---

Kinesis 是 AWS 的一项用于收集实时流数据的云服务，类似于 Kafka。Kinesis 收集到的数据可以用于多个方面，例如存到 S3，发到 EMR 作进一步数据分析等等。

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

**Kinesis Client Library (KCL)**
Kinesis Client Library 是一个用于编写 Consumer 应用的库，简化读取流的操作并且具有容错性。
