---
layout:     post
title:      "Spark Streaming 常见操作"
date:       2018-08-01
author:     "Ink Bai"
catalog:    true
header-img: "http://ox2ru2icv.bkt.clouddn.com/image/post/spark-stream-operations.jpg"
tags:
    - Spark
---

> DStream 的转换操作与 RDD 的差不多，简单的像是 map，flatMap，repartition 我们就不讲了，我们讲几个关键特殊的。

## DStream 的转换操作
**UpdateStateByKey 操作**

**transform 操作**
`transform` 的作用就是弥补 DStream 函数的不足，会将一个 `RDD-to-RDD` 的函数作用在 DStream 上，这样我们就可以将适用于 RDD 的所有函数都应用在 DStream 上。例如我们想实时过滤掉数据中的广告记录，过滤的基础集是之前提前计算好的广告信息数据，那么可以通过如下方式实现：

```scala
val spamInfoRDD = ssc.sparkContext.newAPIHadoopRDD(...) // RDD containing spam information

val cleanedDStream = wordCounts.transform { rdd =>
  rdd.join(spamInfoRDD).filter(...) // join data stream with spam information to do data cleaning
  ...
}
```

注意在每个批次都会调用这个函数，所以在此之间我们可以对这个提前计算的 RDD 进行一些 RDD 操作，例如重分区或者广播等。

**窗口操作**
Spark Streaming 也支持窗口计算，即可以把一个转换作用在滑动的窗口数据上，如下图所示：

![](https://spark.apache.org/docs/latest/img/streaming-dstream-window.png)

上图显示的是一个跨度为 3，每次滑动步数为 2 窗口，一个窗口操作必须指定两个参数：

- 窗口长度：窗口的持续时间（图中是 3）
- 滑动间隔：每次窗口操作之间的间隔（图中是 2）

这两个参数必须是 Spark Streaming 批处理间隔的倍数（图中是 1）。

用一个实例来讲解一下，前面的入门例子我们会用 `reduceByKey` 统计每个批次相同单词的数量，现在我们想每 2 秒统计一次过去 4 秒相同单词的数量应该怎么做呢？需要用到 `reduceByKeyAndWindow` 操作。

```scala
// Reduce last 4 seconds of data, every 2 seconds
val windowedWordCounts = pairs.reduceByKeyAndWindow((a:Int,b:Int) => (a + b), Seconds(4), Seconds(2))
```

下面是一些常用的窗口操作：

转换|含义
-|-
window(windowLength, slideInterval)|返回一个新的窗口 DStream。
countByWindow(windowLength, slideInterval)|返回滑动窗口中所包含元素的数量。
reduceByWindow(func, windowLength, slideInterval)|通过对所有元素依次进行聚合，返回一个包含单个元素的 DStream。从 (K, V) 集合 -> (K, V)
reduceByKeyAndWindow(func, windowLength, slideInterval, [numTasks])|(K, V) 集合 -> (K, V) 集合，根据键进行聚合。注意这个函数还有个 `numTasks` 参数，如果不设置的话并发任务数使用 Spark 默认配置（local 模式是 2，集群模式是 `spark.default.parallelism` 配置的数目），设置了的话就使用这个值。
reduceByKeyAndWindow(func, invFunc, windowLength, slideInterval, [numTasks])|
countByValueAndWindow(windowLength, slideInterval, [numTasks])|



## DStream 的输出操作
