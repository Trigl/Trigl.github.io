---
layout:     post
title:      "Flink on YARN 部署详解（II）"
subtitle:   "启动 JobManager 的过程"
date:       2022-02-09
author:     "Ink Bai"
catalog:    true
header-style: "text"
tags:
    - Flink
---

> 上文主要介绍了 YARN 的架构，一个 Flink 任务真正运行到 YARN 上可以拆分成以下两步：
>
>1. 启动 Flink 集群，其实就是启动 JobManager。
>2. 提交 Flink Job，JobManager 内会启动该任务的 JobMaster 组件，同时在分配的 executor 节点上启动 TaskManager。
>
>本文介绍在 YARN 上启动 Flink 集群的过程，由于 session 模式下启动 Flink 集群和提交运行任务这两步是分离的，为了讲解更加清晰，接下来基于 session-on-YARN 模式进行介绍。

Flink-on-YARN 模式下启动 Flink 集群具体又可分为以下几步：

- Client 端请求在 YARN 上启动一个 ApplicationMaster（不清楚 YARN 组件的传送到这块复习：[YARN 架构](/2022/02/08/flink-on-yarn-1/#架构)）
- 在 YARN AM 内启动 JobManager

## Client 端请求启动 YARN AM
首先在 Flink 目录下执行以下命令启动集群：

```bash
bin/yarn-session.sh -d
```

查看上面的脚本中我们可以看到入口类是 `org.apache.flink.yarn.cli.FlinkYarnSessionCli`

![](/img/content/flink-shell.jpg)

这个入口类的主要作用是启动 YARN 的 ApplicationMaster，这个任务就是启动 JobManager，最核心的代码是 `FlinkYarnSessionCli#run` 内的以下两行：

```java
final YarnClusterDescriptor yarnClusterDescriptor = (YarnClusterDescriptor) yarnClusterClientFactory.createClusterDescriptor(configuration);
clusterClientProvider = yarnClusterDescriptor.deploySessionCluster(clusterSpecification);
```

先看第一行这个类 `YarnClusterDescriptor`，它是在 YARN 上部署 JobManager 的核心类：

![](/img/content/yarn-cluster.jpg)

第二行就是在 YARN 上启动 Flink session 集群，当然里面具体实现实际上是启动 YARN 的 AppMaster，启动时需要指定一个 YARN AM 上的启动类，session 模式下这个类是 `org.apache.flink.yarn.entrypoint.YarnSessionClusterEntrypoint`，这个类也就是 session 模式下启动 Flink JobManager 的入口类。

## YARN AM 上启动 JobManager
向 YARN 提交了启动 AM 的请求之后，AM 内就开始运行我们指定的入口类 `YarnSessionClusterEntrypoint`，注意虽然看起来是启动一个 cluster，但其实此时启动的只是进程 JobManager，后续 job 提交的以后还会启动 TaskManager。

代码跟踪下去可以看到主要逻辑在 `ClusterEntrypoint#runCluster` 内，主要逻辑如下：

```java
private void runCluster(Configuration configuration, PluginManager pluginManager) throws Exception {
	synchronized (lock) {

		initializeServices(configuration, pluginManager);

		// write host information into configuration
		configuration.setString(JobManagerOptions.ADDRESS, commonRpcService.getAddress());
		configuration.setInteger(JobManagerOptions.PORT, commonRpcService.getPort());

		final DispatcherResourceManagerComponentFactory dispatcherResourceManagerComponentFactory = createDispatcherResourceManagerComponentFactory(configuration);

		clusterComponent = dispatcherResourceManagerComponentFactory.create(
			configuration,
			ioExecutor,
			commonRpcService,
			haServices,
			blobServer,
			heartbeatServices,
			metricRegistry,
			archivedExecutionGraphStore,
			new RpcMetricQueryServiceRetriever(metricRegistry.getMetricQueryServiceRpcService()),
			this);
	}
}
```

首先会初始化一些服务，这些服务有包括如下：

- RPC
- 线程池
- HA
- BLOB server
- 指标采集服务
- ExecutionGraph store

之后就是创建 DispatcherResourceManagerComponent，这个类其实是多个组件的集合，主要创建如下几个组件：

- 首先创建并且启动 WebMonitorEndpoint，这个组件提供 Restful 请求响应以及 Web 页面访问
- 之后创建并启动 DispatcherRunner，这个接口包装了真正的组件 Dispatcher，Dispatcher 用来接收作业、提交作业、生成 JobManager 来执行作业，并且可以故障恢复作业，同时还可以查询 Flink Session 集群的信息
- 最后创建并启动 ResourceManager，顾名思义，用于资源的管控

启动了这些核心组件之后，Flink 的 JobManager 就启动完成了，已经做好了接客的准备（处理新提交的任务）。

其实 JobManager 上除了上面的组件还有一个非常重要的组件是 JobMaster，这个组件会在任务提交以后才启动，负责管理单个 JobGraph 的执行。Flink 集群中可以同时运行多个作业，每个作业都有自己的 JobMaster。（JobManager 和 JobMaster 的区别详见：[Flink JobManager](/2021/08/26/flink-architecture/#JobManager)）

另外，启动 JobManager 时的这些服务或者组件，每一个都可以单独拎出来做一个专题，后续有时间可能会开坑写一写，本文不做更多赘述。
