---
layout:     post
title:      "Flink CDC 介绍"
date:       2021-06-09
author:     "Ink Bai"
catalog:    true
header-img: "/img/post/flink-cdc.jpg"
tags:
    - Akka
---

> [flink-cdc](https://github.com/ververica/flink-cdc-connectors) 提供了基于 flink 的 cdc 方案，今天做一个简单介绍。

## 术语解释
CDC：Change data capture

DML：data manipulation language，对数据进行增删改的一些操作

DDL：data definition language，数据库定义

BINLOG：binary log，mysql二进制日志文件，顺序写顺序读，性能高，mysql中会记录最新的文件名和位置。

GTID：全局事务ID（Global Transaction ID），是一个已提交事务的编号，并且是一个全局唯一的编号，主从集群情况下基于GTID恢复和复制会变得非常简单。

Kafka Connect：用于连接Kafka和其他系统，类似于flink connect，但是可部署。

InnoDB MVCC：multi-versioned concurrency control，多重版本控制，这是为了解决高并发情况下一致性读数据的问题。当事物隔离级别设置为repeated read时，开启事物后的第一个select语句那个时间点会形成一个snapshot，此后的select查询结果都是一致的。这个等价于START TRANSACTION WITH CONSISTENT SNAPSHOT语句。

## 常见CDC框架对比
|组件|Canal|Maxwell|Debezium|
|-|-|-|-|
|开源方|阿里|zendesk|redhat|
|开发语言|Java|Java|Java|
|支持数据库|Mysql|Mysql|MongoDB、MySQL、PostgreSQL、SQL Server 、Oracle( 孵化)、DB2( 孵化)、Cassandra( 孵化)|
|是否支持解析DDL同步|是|是|是|
|是否支持HA|是|需定制|基于kafka-connector|
|社区活跃（2021-06-09）|star：19.4k<br>release：2021-04-19|star：2.8k<br>release：2021-06-03|star：4.8k<br>release：2021-05-28|
|文档|中文|英文|英文，官方文档非常详细|
|MQ集成|RocketMQ、Kafka|Kafka|Kafka|

## Debezium 架构
Debezium有三种常见CDC架构：

1、以插件的形式，部署在 Kafka Connect 上
![](/img/content/debezium-architecture.png)
中间的部分是 Kafka Broker，Kafka Connect 是单独的服务，需要下载 debezium-connector-mysql 连接器，解压到服务器指定的地方，然后在 connect-distribute.properties 中指定连接器的根路径即可使用。

2、Debezium Server
![](/img/content/debezium-server-architecture.png)
可以监控两种DB类型的数据变化事件，写入到三种类型的sink中，更改配置即可。

3、内嵌在应用程序里
嵌入引擎的优点是不需要运行在 kafka connect 上，也不运行在server上，而是作为一个库嵌入到自己的应用中。
flink-cdc-connector就是使用了embedded engine，整个架构不需要存储层，可以直接从db检测到change date，发送到对应的各个终端。

## 看源码分析Flink-CDC的过程
#### 创建并启动 Flink 的 SourceFunction
SourceFunction 类如下，目前不能并行执行，**每个库的每个表都会建立一个连接**。
![](/img/content/image2021-6-9_18-19-3.png)

通过Builder模式创建 SourceFunction，相关成员变量如下：
![](/img/content/image2021-6-9_18-38-9.png)

其中StartupOptions是指定从哪里开始同步表，有五种：
- INITIAL：同步表的所有内容，包括两部分，先通过Snapshot同步历史全量数据，再通过binlog同步增量数据
- EARLIEST_OFFSET：从binlog最早的位置同步
- LATEST_OFFSET：从binlog最新的位置同步
- SPECIFIC_OFFSETS：从指定binlog位置同步
- TIMESTAMP：同步binlog在TIMESTAMP时间之后的数据

问题：从社区看，后两个参数目前不可用，想要基于固定位点回溯数据可能需要改Debezium源码。
![](/img/content/image2021-6-10_6-6-8.png)

DebeziumDeserializationSchema 是将从 Debezimu 拿到的 SourceRecord 记录转化为可以被 Flink 处理的记录。

DebeziumSourceFunction 类最核心的成员变量其实就是需要在CK时保存的state如下：
![](/img/content/image2021-6-9_19-51-53.png)

DebeziumSourceFunction run 方法的核心逻辑主要如下：

```Java
@Override
public void run(SourceContext<T> sourceContext) throws Exception {
    // 1、配置properties，主要是Debezium使用
    properties.setProperty("name", "engine");
    properties.setProperty("offset.storage", FlinkOffsetBackingStore.class.getCanonicalName());
    if (restoredOffsetState != null) {
        // restored from state
        properties.setProperty(FlinkOffsetBackingStore.OFFSET_STATE_VALUE, restoredOffsetState);
    }


    // 2、new一个consumer类，用来消费Debezium传过来的数据
    this.debeziumConsumer =
            new DebeziumChangeConsumer<>(
                    sourceContext,
                    deserializer,
                    restoredOffsetState == null, // DB snapshot phase if restore state is null
                    this::reportError,
                    dbzHeartbeatPrefix);

    // 3、创建并运行DebeziumEngine
    this.engine =
            DebeziumEngine.create(Connect.class)
                    .using(properties)
                    .notifying(debeziumConsumer)
                    .using(OffsetCommitPolicy.always())
                    .using(
                            (success, message, error) -> {
                                if (!success && error != null) {
                                    this.reportError(error);
                                }
                            })
                    .build();
    executor.execute(engine);

    // 4、graceful stop
}
```

这个流程中，我们最关心的其实就是一个问题：如何通过flink保存我们当前读取到的数据库位置信息？也就是如何进行CK，而CK又分为全量和增量的情况。

关键实现就是 DebeziumChangeConsumer#handleBatch

flink一般处理record和进程CK是在一个线程中同步执行，这里DebeziumChangeConsumer类是在DebeziumEngine的线程中执行的，与CK线程不一样，需要加锁。

可以看到全量拉数据过程中，一直会持有CK锁，也就是全量时无法进行CK。

```Java
// 批量处理数据
@Override
public void handleBatch(
        List<ChangeEvent<SourceRecord, SourceRecord>> changeEvents,
        DebeziumEngine.RecordCommitter<ChangeEvent<SourceRecord, SourceRecord>> committer)
        throws InterruptedException {
    this.currentCommitter = committer;
    this.processTime = System.currentTimeMillis();
    try {
        for (ChangeEvent<SourceRecord, SourceRecord> event : changeEvents) {
            SourceRecord record = event.value();

            // 过滤掉heartbeat事件，但是仍然需要记录offset
            if (isHeartbeatEvent(record)) {
                // keep offset update
                synchronized (checkpointLock) {
                    debeziumOffset.setSourcePartition(record.sourcePartition());
                    debeziumOffset.setSourceOffset(record.sourceOffset());
                }
                // drop heartbeat events
                continue;
            }

            deserialization.deserialize(record, debeziumCollector);

            // isInDbSnapshotPhase，处于全量db snapshot阶段，此时全程都要获取锁，通过UNSAFE类拿锁
            if (isInDbSnapshotPhase) {
                if (!lockHold) {
                    MemoryUtils.UNSAFE.monitorEnter(checkpointLock);
                    lockHold = true;
                    LOG.info(
                            "Database snapshot phase can't perform checkpoint, acquired Checkpoint lock.");
                }
                if (!isSnapshotRecord(record)) {
                    MemoryUtils.UNSAFE.monitorExit(checkpointLock);
                    isInDbSnapshotPhase = false;
                    LOG.info(
                            "Received record from streaming binlog phase, released checkpoint lock.");
                }
            }

            // emit the actual records. this also updates offset state atomically
            emitRecordsUnderCheckpointLock(
                    debeziumCollector.records, record.sourcePartition(), record.sourceOffset());
        }
    } catch (Exception e) {
        LOG.error("Error happens when consuming change messages.", e);
        errorReporter.reportError(e);
    }
}


/**
 * Takes a snapshot of the Debezium Consumer state.
 *
 * <p>Important: This method must be called under the checkpoint lock.
 */
public byte[] snapshotCurrentState() throws Exception {
    // this method assumes that the checkpoint lock is held
    assert Thread.holdsLock(checkpointLock);
    if (debeziumOffset.sourceOffset == null || debeziumOffset.sourcePartition == null) {
        return null;
    }

    return stateSerializer.serialize(debeziumOffset);
}
```

#### Debezium 从数据库拉数据的过程分析
上面 Flink Source 的代码中，实际上是通过 EmbeddedEngine 拉的数据，这个类就是最核心的引擎类，做了拉取数据，保存状态，断点恢复等的工作。

概括一下就是两步：
- 通过SnapshotReader全量拉数据
- 通过BinlogReader增量拉数据

那么如何区分全量和增量数据呢，记录里面会有字段进行标识：
![](/img/content/image2021-6-10_2-8-19.png)

#### SnapshotReader全量拉数据过程
|步骤|操作|
|-|-|
|1|加全局锁/表级锁|
|2|在重复读的隔离级别下，开启事务，得到mvcc的一致性snapshot<br>SET TRANSACTION ISOLATION LEVEL REPEATABLE READ<br>START TRANSACTION WITH CONSISTENT SNAPSHOT|
|3|读取最新binlog文件名、位置、gtid值|
|4|查询库表schema变化|
|5|释放锁|
|6|select *的方式按批拉取记录，每个记录设置snapshot标签|
|7|最后一条记录设置last snapshot标签|
|8|提交或者回滚事务|

步骤6的流式读取是通过下面方式实现的：
![](/img/content/image2021-6-10_5-48-32.png)

存在问题1：无锁情况下，步骤2和步骤3之间如果有其他并发写操作，会丢失数据
解决：步骤3要放在步骤2之前，数据可能会重复，但不会丢失。

存在问题2：Debezium 对 mysql 支持，而mariadb在 gtid 的机制上并不是完全兼容 mysql
Debezium 会去检测数据库是否开启了 gtid，检测方式为：
![](/img/content/image2021-6-10_5-3-8.png)

Mysql gitd默认关闭，系统变量里有一个 GTID_MODE，设置为 on 就打开了。
而MariaDB 从 10.0.2 开始，GTID 是默认自动打开的，但是这里代码的结果返回的是没有打开gtid。

## 存在的问题
1、不加锁全量拉的过程，全量拉取和后续binlog读取之间的临界点如何处理，会有数据重复或丢失
如果从头同步一张表，分为两部分，首先是获取表snapshot，通过select *读全量；然后读binlog。
获取表snapshot会通过：设置可重复读+开启事务保证一致性。
1. 设置可重复读：SET TRANSACTION ISOLATION LEVEL REPEATABLE READ
2. 开启事务：START TRANSACTION WITH CONSISTENT SNAPSHOT

查看Debezium源码发现它获取全量snapshot的过程是
1. 设置可重复读
2. 开启snapsht的一致性事务
3. 查询当前binlog位置并且保存

注意这三个操作如果不是原子操作，数据可能会丢。Debezium里面有一个参数专门指定这个过程是否加锁：
![](/img/content/image2021-6-8_16-2-48.png)

如果 snapshot.locking.mode 设置为 none，也就是不加锁，在第2步和第3步如果有write行为，就会导致数据丢失或者重复。

Update：Debezium 5.1.2.Final 版本已经修复了这个问题。

2、首次全量拉数据是否要分配很大内存？如果是流式拉是怎么实现的？
可以流式拉取，但是有一个小bug，不影响使用。
https://github.com/ververica/flink-cdc-connectors/issues/98

3、全量拉取如果DB crash，重建连接，是否会丢失数据？
应该会抛连接不上的错误，如果是这样flink任务会直接挂掉重新拉数据

4、增量读取binlog时如果DB crash，是否能够正常恢复？
基于binlog的文件和位置做CK的话，需要binlog文件所在的从库恢复之后flink才能正常拉数据，如果通过gtid做恢复没有问题。

5、ddl 变更场景如何处理？
目前flink-cdc-connector不支持ddl的同步：https://github.com/ververica/flink-cdc-connectors/issues/175
如果业务表的schema发生了改变，需要自己做些兼容或者告警。

6、mariadb 和 mysqldb 关于 gtid 的区别
mariadb在 gtid 的机制上并不是完全兼容 mysql，设置gtid的方式不一样，获取gtid也不一样，检测gtid的机制也不一样
需要更改Debezium代码

7、MVCC 在超大表、频繁读写情况下，事务可能会断掉，拉数据会异常，待具体测试

8、serverId 要保证随机分配

## 总结
flink-cdc 可以利用 flink 的优势来实现 cdc 方案，而且如果使用 flink sql cdc 更可以大大减少代码量，但是业界没有太多实践，如果要用于生产环境，可能需要做一些二次开发或者改造工作，做大量 benchmark 来验证稳定性。
