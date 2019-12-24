---
layout:     post
title:      "「译」当调用 Kafka Consumer 的 poll 方法时发生了什么？"
subtitle:   "Kafka Consumer 初始化和首次拉取数据的原理"
date:       2019-12-20
author:     "Ink Bai"
catalog:    true
header-img: "/img/post/kafka-consumer.jpg"
tags:
    - Kafka
---

> 最近看到一篇 Kafka Consumer poll 源码解析的外文讲的比较清晰透彻，这里翻译学习一下，原文：[What happens when you call poll on Kafka Consumer?](https://chrzaszcz.dev/2019/06/16/kafka-consumer-poll/)

## Consumer 使用示例
只需以下几步就可以使用 kafka Consumer：

- 创建 consumer 并配置
- 订阅 topics
- 以某种方式循环拉取消息

示例代码如下：

```kotlin
override fun run() {
    val kafkaConsumer = createKafkaConsumer()
    kafkaConsumer.subscribe(listOf(TOPIC))
    kafkaConsumer.use { fetchContinuously(kafkaConsumer) }
}
```

让我们详细看一下这几步做了什么。

## 当你创建 consumer 时发生了什么？
使用 consumer 之前必须先创建 consumer，创建时必须设置几个配置项，让我们看一下具体如何创建 consumer：

```kotlin
private fun kafkaConsumer(): KafkaConsumer<String, String> {
    val properties = Properties()

    properties["bootstrap.servers"] = "localhost:9092"
    properties["group.id"] = "kafka-example"
    properties["key.deserializer"] = "org.apache.kafka.common.serialization.StringDeserializer"
    properties["value.deserializer"] = "org.apache.kafka.common.serialization.StringDeserializer"

    return KafkaConsumer(properties)
}
```

这里我们配置了 4 个属性，也是必须配置的 4 个属性，如果不配置的话就无法创建 consumer。

- **bootstrap.servers** - consumer 进行通信和获取集群配置的 Kafka 集群地址。
- **group.id** - 给多个 consumer 指定相同的值，从而可以在这些 consumer 之间进行负载均衡。这里我们使用的是 *kafka-example*。
- **key.deserializer** - 每一个从 Kafka broker 拉取的记录本质上是一组 bytes，所以你必须指定如何解码这些 bytes。这个选项指定了记录中 key 的解码方式，这里我们使用了 `StringDeserializer` 从而可以把 bytes 解码为字符串（默认编码是 UTF-8）
- **value.deserializer** - 和 `key.deserializer` 类似，只不过解码的是记录中的 value 部分。

现在我们已经创建了一个 consumer 的实例，那么 consumer 的构造器到底做了什么呢？其实吧，什么也没做，我们仅仅创建了一些对象，除了一些验证逻辑外什么也没发生。

## 当你订阅 topic 的时候发生了什么？
创建 consumer 之后，接下来必须要做的事情就是订阅 topic 的集合，前面的例子里我们这样订阅 topics：

```kotlin
kafkaConsumer.subscribe(listOf(TOPIC))
```

那么这里内部发生了什么呢？nothing！仅仅是 set 了一些值而已。

到目前为止，我们仅仅是创建了一个 consumer，订阅了几个 topic，其他什么都没有发生。

## 当你第一次拉取数据时发生了什么？
现在就开始有点意思了！废话不多说，直接撸 Kafka Consumer 的源码（version：2.2.0，可以查看 [Github](https://github.com/apache/kafka/blob/2.2/clients/src/main/java/org/apache/kafka/clients/consumer/KafkaConsumer.java#L1179)）。出于方便阅读的考虑，这里我仅留下了需要重点关注的部分。

```java
private ConsumerRecords<K, V> poll(final Timer timer, final boolean includeMetadataInTimeout) {
    acquireAndEnsureOpen();
    try {
        if (this.subscriptions.hasNoSubscriptionOrUserAssignment()) {
            throw new IllegalStateException("Consumer is not subscribed to any topics or assigned any partitions");
        }

        do {
            client.maybeTriggerWakeup();

            if (includeMetadataInTimeout) {
                if (!updateAssignmentMetadataIfNeeded(timer)) {
                    return ConsumerRecords.empty();
                }
            } else {
                while (!updateAssignmentMetadataIfNeeded(time.timer(Long.MAX_VALUE))) {
                    log.warn("Still waiting for metadata");
                }
            }

            final Map<TopicPartition, List<ConsumerRecord<K, V>>> records = pollForFetches(timer);
            if (!records.isEmpty()) {
                if (fetcher.sendFetches() > 0 || client.hasPendingRequests()) {
                    client.pollNoWakeup();
                }

                return this.interceptors.onConsume(new ConsumerRecords<>(records));
            }
        } while (timer.notExpired());

        return ConsumerRecords.empty();
    } finally {
        release();
    }
}
```

所以这里到底做了什么？具体分析一下：

- **Line 2** - 由于 consumer 内部是非线程安全的，所以必须要确保只有一个线程访问 consumer，因此需要在这获取锁。一旦多个线程同时访问这个方法，那么就会报错。
- **Line 4** - Consumer 验证自己是否有任何订阅信息。如果没有指定任何 topics，那么拉取数据也就变成无稽之谈了。
- **Line 8** - 开始拉取记录的循环，直到拉取超时或者接收到了一些记录。
- **Line 9** - 可以在拉取的过程中中断 consumer，比如你设置了拉取的超时时间以后，超过了超时时间就会抛出异常。
- **Line 11** - 获取元数据添加超时机制，注意这个参数建议设置为 true，如果设置为 false 就会一直去同步获取元数据，极端情况可能就一直卡住了，而设置为 true 的话，就会先去尝试更新元数据信息，如果更新失败会立刻返回空记录，结束 poll 过程。获取元数据的方法 `updateAssignmentMetadataIfNeeded` 后文细聊。
- **Line 21** - Consumer 从 kafka 中拉取记录。
- **Line 22** - 这里做了一个优化，如果在上面拉取时成功返回了一些记录，那么 consumer 就会提前发送下一次的拉取请求，这样当你的应用在处理刚刚拉取的新记录的时候，consumer 也同时在后台为你拉取好了下一次要用的数据，避免了 IO 阻塞。
- **Line 27** - Consumer 通过 interceptor chain 传递已经拉取的记录，interceptor 是一种插件化的，你可以通过 interceptor 很方便的对输入的记录做一些更改，一般用于日志和监控。

poll 方法的整个过程做了很多事情，但是总结下来主要是两点：

- 同步 Consumer 和 Kafka Cluster 的元数据信息 - `updateAssignmentMetadataIfNeeded` 方法
- 拉数据 - `pollForFetches` 方法

现在让我们看一下 `updateAssignmentMetadataIfNeeded` 的具体实现！

## 更新分区信息
`updateAssignmentMetadataIfNeeded` 方法是比较简单的，因为实际上它把要干的活都委派给了 coordinator。

```java
boolean updateAssignmentMetadataIfNeeded(final Timer timer) {
    if (coordinator != null && !coordinator.poll(timer)) {
        return false;
    }

    return updateFetchPositions(timer);
}
```

主要有两点：

- 同步更新 coordinator - 确保我们的 consumer group 的 coordinator 是最新的。
- 更新拉取的位移 - 确保当前 consumer 分配的分区更新其相应的拉取位移，如果没有更新到的话，consumer 就会使用 `auto.offset.reset` 来更新分区的拉取位移（设置为最早位移、最近位移或者抛错）。

更新位移的逻辑相当简单明了，所以这里我们着重看一下如何更新 coordinator。直接看一下具体实现，coordinator 的 poll 方法到底做了什么？代码如下（为了可读性做了一些简化），源代码见[这里](https://github.com/apache/kafka/blob/2.2/clients/src/main/java/org/apache/kafka/clients/consumer/internals/ConsumerCoordinator.java#L310)。

```java
public boolean poll(Timer timer) {
    invokeCompletedOffsetCommitCallbacks();

    if (subscriptions.partitionsAutoAssigned()) {
        pollHeartbeat(timer.currentTimeMs());
        if (coordinatorUnknown() && !ensureCoordinatorReady(timer)) {
            return false;
        }

        if (rejoinNeededOrPending()) {
            if (subscriptions.hasPatternSubscription()) {
                if (this.metadata.timeToAllowUpdate(time.milliseconds()) == 0) {
                    this.metadata.requestUpdate();
                }

                if (!client.ensureFreshMetadata(timer)) {
                    return false;
                }
            }

            if (!ensureActiveGroup(timer)) {
                return false;
            }
        }
    } else {
        if (metadata.updateRequested() && !client.hasReadyNodes(timer.currentTimeMs())) {
            client.awaitMetadataUpdate(timer);
        }
    }

    maybeAutoCommitOffsetsAsync(timer.currentTimeMs());
    return true;
}
```

发生了什么呢？

- **Line 4** - 检查 consumer 是使用自动提交位移还是手动提交位移，这里我忽略了手动提交位移的逻辑，主要讲一下自动提交位移。
- **Line 5** - 检查心跳线程的状态并且报告这次的 poll 请求。当首次 poll 请求时还没有创建心跳线程，所以这里其实什么也没有做。但是之后请求时，就会检测心跳线程的状态和报告 poll 请求，如果有问题会抛出错误。

---

你可能会好奇为什么 consumer 需要报告它的请求，实际上调用 poll 方法去拉取数据是你的应用自己的事，然后 Kafka 并不是很信任你，它可能怀疑你在尸位素餐。所以作为预防，consumer 会记录你多久调用一次 poll 方法，一旦这个间隔超过了指定值 `max.poll.interval.ms`，那么当前 consumer 就会从消费者组中离开，这样其他 consumer 就可以接盘当前 consumer 的工作了。

---

- **Line 6** - 这里非常重要！这是我们首次和集群建立通信的地方！`ensureCoordinatorReady` 方法会连接到 `bootstrap.servers` 的一个节点并且拉取整个集群的结构，获取到当前消费者组的 coordinator（通过设置的 `group.id`），然后与该 coordinator 建立连接。如果一切顺利的话那么现在我们已经和**我们自己**的 coordinator 成功会师，可以继续进行下一步了。
- **Line 10** - 检查当前 consumer 是否需要加入 group。事实上 consumer 正是在调用 poll 方法的时候才加入到 group 里面的，如果你更改了 topic 的注册信息（如有 consumer 退出了当前 group），那么当前 consumer 就需要再次重新加入 group。
- **Line 11** - 支持注册 topic 的时候可以使用正则匹配。例如你指定了要订阅的 topic 的正则表达式是 `my-kafka-topic-*`，那么当前 consumer 就会注册到任何匹配这个表达式的 topic。一般来说我们不经常使用正则表达式进行注册，忽略这种情况。
- **Line 21** - 正式加入到 group 中！

尽管 coordinator 的 poll 方法有很多逻辑，但是主要就两点：

- 和我们的 coordinator 建立连接
- 加入当前消费者组

关于 `ensureActiveGroup` 方法，有一个比较重要的细节，前文提到的心跳线程就是在这里被创建的：

```java
boolean ensureActiveGroup(final Timer timer) {
    if (!ensureCoordinatorReady(timer)) {
        return false;
    }

    startHeartbeatThreadIfNeeded();
    return joinGroupIfNeeded(timer);
}
```

一旦这个方法执行成功，那么 consumer 就彻底完成了初始化，随时可以开始拉取数据了！
## 总结
上面我们大致浏览了一下 Kafka Consumer 的源码，研究了第一次调用 poll 方法的原理。现在让我们用时序图总结一下整个过程，如图所示，整个过程就是 consumer 首先获取集群的结构，然后通过 `group.id` 找到消费者组的 coordinator，之后请求加入到消费者组中，开始一个心跳线程，初始化位移，最终拉取数据。

![](/img/content/consumer-first-poll.png)

在研究 Kafka Consumer 的首次 poll 过程中我们可以得到那些结论？

- 初始化一个 consumer 和订阅 topics 不会创建任何连接或线程。
- 每一次的 poll 操作都会确保 consumer 得到初始化，也就是说创建必要的线程、与集群连接、加入到消费者组等一些列操作都是在 poll 中进行的。
- Consumer 是非线程安全的 - 不能同时在多个线程中调用 consumer 的方法，否则会报错。
- 你必须每隔一段时间就调用一次 poll 方法，确保它自身的存活性以及与 Kafka 集群的连通性。
- 有一个心跳线程用于告知 Kafka 集群当前 consumer 是否存活，这个线程也是在调用 poll 方法时建立的。
