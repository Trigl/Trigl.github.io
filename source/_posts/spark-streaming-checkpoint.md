---
layout:     post
title:      "Spark Streaming Checkpoint"
date:       2018-07-31
author:     "Ink Bai"
catalog:    true
header-img: "http://ox2ru2icv.bkt.clouddn.com/image/post/spark-stream-checkpoint.jpg"
tags:
    - Spark
---

> 一个 Streaming 应用是一个 007 特工，需要保证 7 * 24 小时的持久运转，因此容错性就极其重要，Spark Streaming 通过在一个具有容错性的存储系统如 HDFS 中设置一些检查信息来从错误中恢复。

## 什么情况下需要设置 checkpoint？
首先我们看一下哪些数据会被 checkpoint，主要有两种类型的数据：

- 元数据：将定义 Streaming 应用如何计算的信息存储到容错的文件系统中，用来从运行该 Streaming 应用的失败节点中恢复过来。源数据包括：
  - Streaming 应用的配置
  - 定义该 streaming 应用的 DStream 操作的集合
  - 正在处理但是尚未完成的批次

- 数据：在 Streaming 中有些转换是无状态的，例如每个批次单独的数据处理；有些转换是有状态的，即本批次的计算需要之前批次的数据，例如窗口操作（后面会讲到）需要结合多个批次的数据。有状态的转换需要依赖于之前批次的 RDDs，这就导致随着时间推移需要维护的依赖链会越来越长，那么恢复的时候势必会很麻烦，所以我们会把那些必须的在此期间生成的 RDDs 阶段性地存到容错性存储中。

从上面讲的我们可以知道什么时候需要设置检查点呢？

**如果你想从出错的节点中恢复 streaming 应用，或者是你的应用中包含有状态的 DStream 操作，那么就需要设置检查点。**

## 如何设置 checkpoint？
首先需要在一个文件系统中定义 checkpoint 目录，然后通过如下设置这个目录：

```scala
streamingContext.checkpoint(checkpointDirectory)
```

这种情况下如何得到 context 呢？使用下面的方式：

```scala
// Get StreamingContext from checkpoint data or create a new one
val context = StreamingContext.getOrCreate(checkpointDirectory, functionToCreateContext _)
```

如果 `checkpointDirectory` 存在，就从 checkpoint 中的数据恢复一个 context，否则新建一个 context。注意 checkpoint 存在一定的开销，所以应该设定一个合适的间隔时间，设置方式如下：

```scala
dstream.checkpoint(checkpointInterval)
```

**如果不设置的话默认是 10 s，一般来说检查点间隔设置为 Spark Streaming 间隔的 5 - 10 倍比较合适。**

## Accumulator 和 Broadcast 在 Checkpoint 中的使用
`Accumulator` 和 `Broadcast` 不能从 checkpoint 中恢复，所以如果想要在应用失败重启后重新实例化这两个对象的话，就需要创建他们的**懒加载单实例**，保证在多线程情况下每次初始化的都是同一对象，具体代码如下：

```scala
/**
  * Use this singleton to get or register a Broadcast variable
  */
object WordBlacklist {
  @volatile private var instance: Broadcast[Seq[String]] = null

  def getInstance(sc: SparkContext): Broadcast[Seq[String]] = {
    if (instance == null) {
      synchronized {
        if (instance == null) {
          val wordBlacklist = Seq("a", "b", "c")
          instance = sc.broadcast(wordBlacklist)
        }
      }
    }
    instance
  }
}

/**
  * Use this singleton to get or register an Accumulator
  */
object DroppedWordsCount {
  @volatile private var instance: LongAccumulator = null

  def getInstance(sc: SparkContext): LongAccumulator = {
    if (instance == null) {
      synchronized {
        if (instance == null) {
          instance = sc.longAccumulator("WordsInBlacklistCounter")
        }
      }
    }
    instance
  }
}

// Get or register the blacklist Broadcast
val blacklist = WordBlacklist.getInstance(rdd.sparkContext)
// Get or register the droppedWordsCounter Accumulator
val droppedWordsCounter = DroppedWordsCounter.getInstance(rdd.sparkContext)
```

## WordCout 实例
官网上有一个实时计算重复单词数量的例子，设置了 checkpoint，同时也使用了 `Accumulator` 和 `Broadcast`，可以直接在本地运行，源码见这里：[RecoverableNetworkWordCount](https://github.com/Trigl/spark-learning/blob/master/src/main/scala/ink/baixin/spark/examples/streaming/RecoverableNetworkWordCount.scala)
