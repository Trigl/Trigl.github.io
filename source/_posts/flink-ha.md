---
layout:     post
title:      "Flink JobManager 高可用详解"
#subtitle:   "YARN 架构 && Flink on YARN 简介"
date:       2022-02-11
author:     "Ink Bai"
catalog:    true
#header-style: "text"
header-img: "/img/archive-bg.jpg"
tags:
    - Flink
---

说到高可用，本质上都是为了尽可能降低系统的不可用时间，对于 Flink 这样的流计算引擎而言，高可用也就意味着实时任务的断流时间尽可能的小，表现在时间上就是低延迟，这对一些时效性要求高的作业来说至关重要。当然，Flink 任务整体的高可用涉及到很多方面，如 JobManager 保证高可用、TaskManager 的故障恢复、甚至 connector 上可以做一些高可用保障，本文主要介绍 JobManager 的高可用，会尝试解答以下几个问题：

> **1. Flink JobManager 的高可用用来解决什么问题？**
**2. 如何配置 JobManager 高可用？**
**3. JobManager 的高可用具体是怎么实现的？**
**4. Flink on YARN 下 JobManager 的高可用有何特殊之处？**
**5. 当前的高可用实现是否存在问题、生产实践中有哪些坑？**

## JobManager 高可用简介
JobManager 的高可用就是用来解决其单点问题，高可用的一般概念指的集群中同时存在多个点，其中一个是 leader 节点，其他的都是 stand-by 节点，当 leader 故障时 stand-by 中的某个节点升级为 leader 继续提供服务，也就是保证了系统的「高可用」，如图：

![](/img/content/jobmanager_ha_overview.png)

当 JobManager 挂掉之后，stand-by 节点可以快速恢复一个新的 JobManager，任务可以被重新拉起。在 per-job 模式下作用可能不是很大，但是在 session 模式下 JobManager 高可用是非常重要的。

Flink 提供了两种具体的高可用实现，分别是基于 ZooKeeper 和基于 Kubernetes，本文主要讲解 Flink on YARN 模式下基于 ZooKeeper 的高可用实现。

## 如何配置？
#### YARN 配置
Flink on YARN 模式下的高可用依赖于 YARN Application Master（AM）的自动恢复，需要在 `yarn-site.xml` 添加如下配置，这个配置指定了 AM 失败之后的最大重启次数：

```
<property>
  <name>yarn.resourcemanager.am.max-attempts</name>
  <value>4</value>
  <description>
    The maximum number of application master execution attempts.
  </description>
</property>
```

#### Flink 配置
`flink-conf.yaml` 添加如下配置：

```
high-availability=zookeeper
high-availability.zookeeper.quorum=localhost:2181
high-availability.storageDir=hdfs:///flink/ha
zookeeper.sasl.disable=true
yarn.application-attempts=2
```

`high-availability.storageDir` 用来指定一个 FileSystem 的路径用于持久化数据，后面会讲到为什么需要配置。
`zookeeper.sasl.disable` 配置是否禁用 ZooKeeper 安全认证，如果你的 ZooKeeper 集群没有安全认证的话，需要把这个禁掉。

## 深入源码：如何实现？
Flink 的高可用服务需要通过下面几个细分服务来实现：

- leader 选举：当 JobManager 出现故障时，需要从其他 stand-by 节点中找一个替补作为新的 JobManager，此时就需要进行 leader 选举，即从 n 个候选者中选出一个领导者
- leader 发现：TaskManager 需要跟 JobManager 地址保持通信，那新的 JobManager 之后，TaskManager 如何感知到呢？这也就是一个服务发现的过程，需要能够检索到当前领导者的地址。而分布式环境下的服务发现需要依赖于能够提供一致性的系统，例如 ZooKeeper。
- 状态持久化：JobManager 恢复需要某些元数据，如作业的 JobGraphs、用户代码 jar、已完成的检查点等，因此需要将这些状态持久化下来。一般这些状态比较大，需要持久化到一个第三方 FileSystem 中，例如 HDFS，前面 `high-availability.storageDir` 就会指定一个持久化路径。

#### 接口设计
Flink 源码中高可用的实现是通过接口 `HighAvailabilityServices` 来直接体现的，可以说这个接口直接暴露了高可用所具有的一切能力，我们来看一下都有哪些方法：

```Java
public interface HighAvailabilityServices extends ClientHighAvailabilityServices {

    LeaderElectionService getResourceManagerLeaderElectionService();

    LeaderElectionService getDispatcherLeaderElectionService();

    LeaderElectionService getJobManagerLeaderElectionService(JobID jobID);

    default LeaderElectionService getClusterRestEndpointLeaderElectionService() {
      return getWebMonitorLeaderElectionService();
    }

    LeaderRetrievalService getResourceManagerLeaderRetriever();

    LeaderRetrievalService getDispatcherLeaderRetriever();

    LeaderRetrievalService getJobManagerLeaderRetriever(JobID jobID, String defaultJobManagerAddress);

    @Override
    default LeaderRetrievalService getClusterRestEndpointLeaderRetriever() {
      return getWebMonitorLeaderRetriever();
    }

    CheckpointRecoveryFactory getCheckpointRecoveryFactory();

    JobGraphStore getJobGraphStore() throws Exception;

    RunningJobsRegistry getRunningJobsRegistry() throws Exception;

    BlobStore createBlobStore() throws IOException;
}
```

可以发现的确提供了开头讲的服务能力：

- leader 选举和 leader 发现：选举和发现是配对存在的，先进行 leader 选举，选举成功之后其他组件才能进行 leader 发现，细分下来 Flink 真正进行 leader 选举和发现的是如下四个组件
  - Resource Manager
  - Dispatcher
  - RestEndPoint
  - JobManager
- 状态持久化
  - CheckpointRecoveryFactory：checkpoint 相关状态
  - JobGraphStore：提交任务的 JobGraph 状态
  - RunningJobsRegistry：任务的状态，例如是处于运行态还是结束态
  - BlobStore：BlobStore 提供了管理下载大文件的能力，比如用户 jar

上面介绍了为高可用功能抽象出来的的接口类，这个接口的 ZooKeeper 实现类就是 `ZooKeeperHaServices`。

#### 基于 ZooKeeper 的 HA 实现
**1. leader 选举**
如果 Flink 集群还没有选出主 JobManager，此时多个从 JobManager 首先会通过 ZooKeeper 选出 leader，选出之后这个 leader 会在 ZooKeeper 上创建一个「临时节点」，并且在这个节点上写入该 leader 的 address 和一个 sessionId，写入这个 address 信息是为了之后进行 leader 发现。临时节点路径如下：

```
/flink/cluster_id_1/resource_manager_lock
/flink/cluster_id_1/job-id-1/job_manager_lock
```

为什么要写入到临时节点呢？leader 节点会与 ZooKeeper 保持一个 session 连接，当这个连接断掉时，临时节点就会被 ZooKeeper 删除。因为当 leader 和 ZooKeeper 无连接时，它就不再是 leader 了，因此删掉其 address 信息，其他节点就不会发现一个错误的 leader 地址了。

除了写入 leader 的 address 之外还会写入一个 sessionId，每个 leader 都会生成一个唯一的 sessionId，可以通过这个 sessionId 区分 leader 是否发生了变化。

**2. leader 发现**
leader 发现也比较简单，它是基于 ZooKeeper 的 Watcher 模式，即可以对 ZooKeeper 的节点进行主动监听，只要节点发生变化，如节点内容变更、节点被删除或者异常信息，都可以通知到 listener。

如需要拿到 JobManager 地址的组件，比如 TaskManager，启动一个 ZooKeeper client 并且监听 JobManager 在 leader 选举写入的临时节点，从而拿到其 address 和 sessionId，之后便可进行通信。若 JobManager 切换了 leader，临时节点上的内容发生变化，此时 TaskManager 也可第一时间被通知到，会拿到新的 JobManager address，保证任务的正常运行。

**3. 持久化数据**
ZooKeeper 的目的是用来解决分布式系统的一致性问题，并不是做存储，一般不能存储太大的数据。所以 HA 要保存的数据，如果比较小会直接保存在 ZooKeeper，如果比较大则会保存在一个 FileSystem 如 HDFS 中，而 ZooKeeper 仅保存文件在 FileSystem 的路径。

## Flink on YARN 模式下的高可用有何不同？
如上所述，一般概念的高可用同时存在多个实例，保证 leader 实例出现问题之后其他「替补」可以及时补上。但是 Flink on YARN 模式下，一个 Flink 集群仅会启动一个 JobManager，不会同时启动多个 JobManager，当 JobManager 出现故障以后也是由 YARN 自动感知并重新启动一个新的 JobManager。

![](/img/content/yarn-restart-am.jpg)

尽管 YARN 恢复重启 JobManager 也需要时间，并不能做到真正意义上的 JobManager 永不宕机，但是得益于 YARN 高效的资源调度和分配能力，这个重启的时间会非常快，所以也可以看作是实现了高可用。这种方式的好处是：

- 日常不需要启动 stand-by JobManager，避免了资源浪费
- 故障时多个 stand-by 节点同时争夺 leader，可能存在并发问题；管理单个 JobManager 比较简单，不容易出问题

## 存在的问题
#### ZooKeeper 的稳定性
当网络抖动、机器繁忙、ZK 集群暂时无响应或运维机器的时候，都可能会导致任务重启。

![](/img/content/b3ef95693bda78d340a93fcef5ba4105.jpg)

任务重启的原因是由于在这些场景发生时，Curator 会将状态设置为 suspended，并且 Curator 认为 suspended 为 Error 状态，从而会释放 leader，JobManager 发现自己不是 leader 后会调用 revokeLeadership，从而造成任务重启。

一个可行的解决办法是升级 Curator 的版本，同时将 connectionStateErrorPolicy 设置为 SessionConnetionStateErrorPolicy。

![](/img/content/9fa06120d2443dc58d8e743df8afedf7.jpg)

上面讨论的是 ZK 抖动的情况，我们再来考虑一下极端情况，ZK 整个都挂了，短期之内无法恢复，这个时候怎么办呢？

此时我们就需要准备作业启动时的降级方案了，在 ZK 不可用时，应该做到作业的启动不受影响，降级到不开启 HA 的启动方式，通过指定 checkpoint 路径进行启动。

除了降级的方案，还有其他提升 HA 可用性的方案，例如在 ZK 异常时我们也可以选择 HDFS 进行 leader 选举和发现，具体见 [ZooKeeper 故障时如何保证 Flink JobManager 的高可用？](/2022/03/11/flink-ha-zk-and-hdfs/)

#### yarn.application-attempts 参数
`yarn.application-attempts` 是指 YARN 可对 AM 进行重启恢复的最大次数，YARN 每次重启 AM 都是一次新的 attempt。需要注意的是 Flink 配置 HA 后这个参数默认设置为 2，这是为什么呢，能不能配置成其他值？

YARN 中，AM 失败次数必须小于这个值才允许被重启，因此如果这个值设置为 1 那就意味着 AM 只要挂掉就无法再重启了。

然后要明白一点，AM 连续失败重启次数是指在一个固定时间间隔内，AM 失败重启了几次，具体实现在 `org.apache.hadoop.yarn.server.resourcemanager.rmapp.RMAppImpl#getNumFailedAppAttempts` 内：

```Java
private int getNumFailedAppAttempts() {
  int completedAttempts = 0;
  long endTime = this.systemClock.getTime();
  // Do not count AM preemption, hardware failures or NM resync
  // as attempt failure.
  for (RMAppAttempt attempt : attempts.values()) {
    if (attempt.shouldCountTowardsMaxAttemptRetry()) {
      if (this.attemptFailuresValidityInterval <= 0
          || (attempt.getFinishTime() > endTime
              - this.attemptFailuresValidityInterval)) {
        completedAttempts++;
      }
    }
  }
  return completedAttempts;
}
```

可以看到，当前时刻往前 `attemptFailuresValidityInterval` 时间段内的 attempt 才会被算到连续重试次数内，Flink 设置的 `attemptFailuresValidityInterval` 默认值是 10s，一般这么短时间 AM 失败重启 1 次，因此可以把 `yarn.application-attempts` 设置成 2.

那么能否设置成大于 2 的值呢？Application 的 AM 挂了之后，YARN 重启 AM 的时候可以选择保留这个 Application 其他的 container，这样可以避免再次申请 container，当参数 `yarn.application-attempts` 设置大于 2 时就可以实现这个效果。听起来很好，但是 Flink 重启 JobManager 之后会恢复所有的 JobMaster，如果此时复用之前的 container，就会在之前的 TaskManager JVM 内重新拉起对应的 task，然而 TaskManager 之前的 task 资源如内存和线程并没有被释放，就会产生资源泄漏问题，如下就是重启 JobManager 多次后 TaskManager 线程数和堆外内存情况：

![](/img/content/tm-thread.jpg)

![](/img/content/tm-direct.jpg)

因此虽然 YARN 提供了复用 container 的策略，但是 Flink 却无法使用，在 session 模式如果 JobManager 发生重启，就会同时重启所有的任务，如果任务数很多的话就存在很大的风险了，这部分可以优化的点是热重启 JobManager，也就是重启 JobManager 不影响 TaskManager。

## 总结
本文首先介绍了 Flink HA 解决的问题和配置方式，然后讲解了实现 HA 需要实现如下相关服务：

- leader 选举和发现
- 持久化恢复 JobManager 需要的信息

接着讲解了基于 ZooKeeper 的 HA 实现，核心是先通过 ZooKeeper 选出 leader 节点，然后再利用其 Watcher 机制进行 leader 发现，通过存储在 ZooKeeper 和外部 FileSystem 中的数据来恢复新的 JobManager。

然后我们说明了 Flink on YARN 模式下 HA 的不同之处：任何时候只会存在一个 JobManager 节点，不会有其他备用的 stand-by 节点，高可用是通过 YARN ResourceManager 快速重启 ApplicationMaster（JobManager）实现的，这种 case 下也就不存在多节点竞争 leader 的问题，也不会有并发问题。

最后又介绍了实际生产环境中可能存在的问题，主要介绍了 ZooKeeper 抖动或者 ZooKeeper 不可用的情况下如何处理。

## Refer
[JobManager 高可用](https://nightlies.apache.org/flink/flink-docs-release-1.14/zh/docs/deployment/ha/overview/)
[京东Flink优化与技术实践](https://cloud.tencent.com/developer/news/738891)
[腾讯基于 Flink 的实时流计算平台演进之路](https://toutiao.io/posts/f0mjql/preview)
[The Number of Maximum Attempts of an Yarn Application in Hadoop Two](http://johnjianfang.blogspot.com/2015/04/the-number-of-maximum-attempts-of-yarn.html)
[yarn关于app max attempt深度解析，针对长服务appmaster平滑重启](https://www.cnblogs.com/yanghuahui/p/4911276.html)
