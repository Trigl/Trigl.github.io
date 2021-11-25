---
layout:     post
title:      "闲聊 Kafka Producer 的设计"
date:       2021-11-23
author:     "Ink Bai"
catalog:    true
header-style: "text"
tags:
    - Kafka
---

> 最近在看 Kafka Producer 的源码，直接讲解源码比较枯燥，本文将结合我们实际生活中的例子初步了解下 Kafka Producer 的设计思路。

## Kafka Producer 怎么用？
Kafka Producer 的用法是很简单的，指定一个 brokerlist 地址，指定一个 topic，然后往这个 topic 发数据就可以了。

```Java
Properties props = new Properties();
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.ByteArraySerializer");

KafkaProducer producer = new KafkaProducer(props);
ProducerRecord<String, byte[]> record = new ProducerRecord<>("test-topic", "hello world".getBytes(StandardCharsets.UTF_8));
producer.send(record, ((metadata, exception) -> {
    if (exception != null) {
        // handle error
    } else {
        // handle success
    }
}));
producer.close();
```

## 如果是你，如何设计？
如果让你来设计 Kafka Producer，你会如何设计？

先看一下我们要达到的目标，说起来非常简单，本质上就是把消息通过网络从 A 传递到 B，但是如果我们把目标细化一下，发现问题没有这么简单：

- 给定一个 topic，这个 topic 有多个分区，我们要实现的是不停地把数据发往这个 topic 的某个分区
- 具体记录应当发往哪个分区，需要有一个分区算法
- 发送的不止是一个 topic，可以有多个
- 发送的效率要高，不能来一条消息就发一条
- 发送完成后要接收 response 确定成功或者失败

可以看到，设计 Kafka Producer 的难点其实是如何高性能发送数据到 Server，它主要耗费的时间和资源就是网络 IO，那么面对 IO 密集型的应用，如何提高性能呢？

- 首先你可能会想到用多线程来优化，可惜它对 IO 密集型应用无能为力
- 如果同样的数据量，从 10 次网络 IO 操作 降低到 5 次网络 IO，性能一定会有提升，如何做到呢，答案就是攒批发送
- 如何攒批？我们实际要发送到 Kafka 的某个 broker 上，因此需要按照 broker 地址来攒批。另外每个 broker 上分布着多个 topic，我们还需要提前按照 topic 维度攒好批。按照 topic 维度攒批并不会降低网络 IO 的次数，这样做是为了省去数据发到 server 后再按照 topic 进行数据聚合的成本，这件事情在 server 端做的成本要比在 producer 端做大很多。

## 发快递！
上面我们基本意淫了一下实现的思路，那 Kafka Producer 真正的设计是怎样的呢？非常幸运，Kafka Producer 的设计和发快递的流程很像，我们可以类比着来讲解。

发快递大体分为这么几个步骤：

- 快递员上门揽件
- 快递被送回仓库
- 仓库内分拣员对快递按照地区进行筛检装箱
- 当某个地区快递量达到一定规模的时候，比如攒够一车皮、一飞机，正式发走

了解了发快递的运转模式，让我们在宏观上看看 KafkaProducer 的设计。首先主线程把要发送的消息按照主题分区进行累积，达到一定数量后，触发发送线程进行发送。为了提高发送的效率，把发往同一个服务器的消息进行归并，一次性发往相应的服务器。

快递中的成员对应到代码中相应的类如下：

- 收件员-KafkaProducer：发送消息的第一步就是调用 KafkaProducer.send() 方法
- 仓库分拣员-RecordAccumulator：负责把消息按照 topic 和分区维度筛选聚合在一起
- 大快递箱-ProducerBatch：每个 ProducerBatch 起始就是一个邮件收拢箱，同一个 topic 和分区的快递箱子放在一起，程序中就是数据结构 Deque<ProducerBatch>
- 运输车-Sender：负责运输，把消息真正发送出去，内部通过 NIO 实现网络传输
- 货车车厢-ClientRequest：用 Sender 发送数据时，其内部实际上是把发往同一个服务器的消息放入 ClientRequest 中。

下图展示了 Kafka Producer 工作的主流程：

![](/img/content/producer-express.jpg)

图中可以看到，有两个线程同时在工作，一个线程负责把消息送往消息站进行分组，另外一个线程负责把消息真正发送出去。

## 总结
以上我们对 Kafka Producer 的设计思路做了一个简单讲解，可以看到优秀的设计也是可以源于生活的，一方面，生活中的实践都经过了时间的验证，一般都是效率最高的，另一方面，基于生活原型的设计，也是更容易理解和写代码的。当然本文只是在一个很浅的层面上了解了一下 Kafka Producer 的整体设计，后面我们会结合源码学习一下 Kafka Producer 的各种精妙设计，一定会让你卧槽不已。
