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
现在就开始有点意思了！废话不多说，直接撸 Kafka Consumer 的源码（version：2.2.0，可以查看 [Github](https://github.com/apache/kafka/blob/2.2/clients/src/main/java/org/apache/kafka/clients/consumer/KafkaConsumer.java#L1179) 的源码）。出于方便阅读的考虑，这里我仅留下了需要重点关注的部分。

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

- **Line 2** - 由于 consumer 内部是非线程安全的，所以必须要确保只有一个线程访问 consumer，因此需要在这获取锁。一旦多个线程同时访问这个方法，那么某些就会报错。
- **Line 4** - Consumer 验证自己是否有任何订阅信息。如果没有指定任何 topics，那么拉取数据也就变成无稽之谈了。
- **Line 8** - 开始拉取记录的循环，直到拉取超时活着接收到了一些记录。
