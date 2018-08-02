---
layout:     post
title:      "Spark Streaming 集成 AWS Kinesis"
date:       2018-08-02
author:     "Ink Bai"
catalog:    true
header-img: "http://ox2ru2icv.bkt.clouddn.com/image/post/spark-kinesis.jpg"
tags:
    - Spark
---

> 关于 AWS Kinesis 的基本信息可以看我的这篇文章：[使用 AWS Kinesis 收集流数据](http://baixin.ink/2018/05/16/collect-data-with-kinesis/)，本文主要讲解 Spark Streaming 如何集成 Kinesis 处理流数据。

## 配置 Spark Streaming 应用
Spark Streaming 集成 Kinesis 的主要代码如下：

```scala
import org.apache.spark.storage.StorageLevel
import org.apache.spark.streaming.kinesis.KinesisInputDStream
import org.apache.spark.streaming.{Seconds, StreamingContext}
import com.amazonaws.services.kinesis.clientlibrary.lib.worker.InitialPositionInStream

val kinesisStream = KinesisInputDStream.builder
    .streamingContext(streamingContext)
    .endpointUrl([endpoint URL])
    .regionName([region name])
    .streamName([streamName])
    .checkpointAppName([Kinesis app name])
    .checkpointInterval([checkpoint interval])
    .initialPositionInStream([initial position])
    .storageLevel(StorageLevel.MEMORY_AND_DISK_2)
    .buildWithMessageHandler([message handler])
```

下面主要讲解一下其中的参数：

**streamingContext**
用于从 Kinesis 中接收并处理数据的 StreamingContext。

**endpoint URL**
AWS Kinesis 的终端节点，还可以从这个值得到区域信息，所有的终端节点可以从 [AWS 区域和终端节点](https://docs.aws.amazon.com/zh_cn/general/latest/gr/rande.html#ak_region) 看到。

**region name**
AWS 区域名称，可以从上面的终端节点得到。

**streamName**
Kinesis stream 的名称。

**Kinesis app name**
每一个从 Kinesis 中接收数据的应用都是一个 consumer，这里指的就是这个应用的名称，必须设置，后面会讲到。

**checkpoint interval**
检查点间隔时间，这里的检查点是指 Kinesis 自身的检查点而非 Spark Streaming 的检查点。为什么要设置检查点呢？由于 Kinesis 是一个实时流消息系统，那么当 Spark Streaming 从中接收数据的时候，必须保证数据的完整性，这就需要定期存储数据流的信息，那么是存在哪里呢？
假设现在我们有一个叫 `spark-kinesis-test` 的应用来接收 Kinesis 数据，当设置了检查点以后，该应用第一次从 Kinesis 中接收数据时，会先在 DynamoDB 中新建一个表名为 `spark-kinesis-test` 的表，之后会定期把接收到的 Kinesis 消息记录如序列号的信息存到这个表中，这样当该应用出现问题重启以后，数据就可以从记录的这个点开始接收，保证数据不会丢失。这个操作是定期做的，检查点间隔时间设置的就是这个值。

**initial position**
读取消息的起始位置，指的是从 Kinesis 流的哪个位置开始读取消息。设置了检查点自然从检查点的位置读取，这个值是说明在没有设置检查点的情况下应该从什么位置开始，目前有两个值：

- InitialPositionInStream.LATEST：从从后一条消息开始读取，显然这种的很可能会造成丢数据。
- InitialPositionInStream.TRIM_HORIZON：从第一条消息记录开始读，显然这样会导致重复处理数据。

**message handler**
一个用来将 Kinesis `Record` 类型转换成任意类型 `T` 的函数。

注意下面几点：

1. Spark Streaming 处理 Kinesis 消息可以实现 `at-least once`。
2. 多个应用可以同时从相同的 Kinesis 流读取消息，DynamoDB 中会保存其分片和序列号信息，这两个信息唯一指定了一条消息。
3. 一般一个 `KinesisInputDStream` 会创建一个 `KinesisRecordProcessor` 线程，所以只会处理 Kinesis 中一个分片的消息。当然如果想同时处理多个分片的消息，只需要创建多个 `KinesisRecordProcessor` 线程即可。所以创建的 `KinesisInputDStream` 的数目不需要大于 Kinesis 的分片数。

## 实践
让我们使用 WordCount 的例子来进行讲解，首先我们需要有一个 AWS 账号，然后创建一个 Kinesis stream。

首先我们要创造一些数据发到 Kinesis 中：

```scala
// Create a partitionKey based on recordNum
val partitionKey = s"partitionKey-$recordNum"

//Create a PutRecordRequest with an Array[Byte] version of the data
val putRecordRequest = new PutRecordRequest().withStreamName(stream)
  .withPartitionKey(partitionKey)
  .withData(ByteBuffer.wrap(data.getBytes()))

// Put the record onto the stream and capture the PutRecordResult
val putRecordResult = kinesisClient.putRecord(putRecordRequest)
```

完整代码见 [KinesisWordProducerASL](https://github.com/Trigl/spark-learning/blob/master/src/main/scala/ink/baixin/spark/examples/streaming/KinesisWordProducerASL.scala)

然后就是使用 Spark Streaming 接收并处理 Kinesis 流数据：

```scala
// In this example, we're going to create 1 Kinesis Receiver/input DStream for each shard,
// This is not a necessity; if there are less receivers/DStreams than the number of shards,
// then the shards will be automatically distributed among the receivers and each receiver
// will receive data from multiple shards.
val numStreams = numShards

// Create the Kinesis DStreams
val kinesisStreams = (0 until numStreams).map { i =>
  KinesisInputDStream.builder
    .streamingContext(ssc)
    .streamName(streamName)
    .endpointUrl(endpointURL)
    .regionName(regionName)
    .initialPosition(new Latest())
    .checkpointAppName(appName)
    .checkpointInterval(kinesisCheckpointInterval)
    .storageLevel(StorageLevel.MEMORY_AND_DISK_2)
    .build()
}
```

完整代码见 [KinesisWordCountASL](https://github.com/Trigl/spark-learning/blob/master/src/main/scala/ink/baixin/spark/examples/streaming/KinesisWordCountASL.scala)

Producer 的输出如下：

```scala
Putting records onto stream spark-kinesis-test and endpoint https://kinesis.cn-north-1.amazonaws.com.cn at a rate of 10 records per second and 5 words per record
Sent 10 records
Sent 10 records
Sent 10 records
Sent 10 records
Sent 10 records
Sent 10 records
Sent 10 records
Sent 10 records
Sent 10 records
Sent 10 records
Totals for the words sent
(Spark,106)
(are,99)
(my,82)
(son,108)
(you,105)
```

Spark Streaming 的输出如下：

```scala
-------------------------------------------
Time: 1533118924000 ms
-------------------------------------------
(are,99)
(son,108)
(my,82)
(Spark,106)
(you,105)
```

可以看到输入的数据和输出的数据结果保持一致。

## 总结
本文主要讲解了如何在 Spark Streaming 中集成 AWS Kinesis 处理实时流数据，讲解了一些基本概念和编程配置，并给出了一个实例。但是可以注意到目前仅实现了消息的 `at-least once`，那么如何实现 `exactly once` 呢？这是我们后续需要研究的内容。
