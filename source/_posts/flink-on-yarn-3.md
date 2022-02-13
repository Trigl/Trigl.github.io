---
layout:     post
title:      "Flink on YARN 部署详解（III）"
subtitle:   "Flink Job 的提交和运行"
date:       2022-02-10
author:     "Ink Bai"
catalog:    true
header-style: "text"
tags:
    - Flink
---

> 上文讲了 Flink JobManager 的启动过程，也就是完成了一个 Flink Cluster 的部署，那么接下来我们看一下一个 Flink Job 的提交运行过程。

Flink Job 提交的大体流程如下：

![](/img/content/flink-on-yarn.png)

这个图画的是早期 Flink 版本的架构，图中的 JobManager 在老的 runtime 框架中是存在的，现在把它看成是 JobMaster 就可以。

我们分成以下两步讲解：

- Client 端提交 Flink Job 到 JobManager
- JobManager 运行提交的 Job

## Client 端提交 Flink Job
提交 Flink Job 的方式有多种，可以通过命令行方式也可以通过 Restful 请求，在 Flink 目录下用命令行方式提交任务如下：

```shell
bin/flink run -d ./examples/streaming/TopSpeedWindowing.jar
```

进入这个 shell 脚本可以看到启动类是 `org.apache.flink.client.cli.CliFrontend`，注意这个类是运行在 Client 端的，此时还没有提交到 Flink 集群上。跟进代码可以看到主要做了两件事：

- 加载配置，如果在 JobManager 机器上提交 Flink Job，会从本地目录找到 YARN 配置文件：

  ```java
	public static File getYarnPropertiesLocation(@Nullable String yarnPropertiesFileLocation) {

		final String propertiesFileLocation;

		if (yarnPropertiesFileLocation != null) {
			propertiesFileLocation = yarnPropertiesFileLocation;
		} else {
			propertiesFileLocation = System.getProperty("java.io.tmpdir");
		}

		String currentUser = System.getProperty("user.name");

		return new File(propertiesFileLocation, YARN_PROPERTIES_FILE + currentUser);
	}
	```

  找到配置文件就可以确定 YARN 的 ApplicationId 了，当然也可以在命令行通过 `--applicationId` 指定。

- 初始化任务运行的上下文环境

  ```java
  StreamContextEnvironment.setAsContext(
    executorServiceLoader,
    configuration,
    userCodeClassLoader,
    enforceSingleJobExecution,
    suppressSysout);
  ```

- 运行 Job Jar 的入口类，这里也就是 `./examples/streaming/TopSpeedWindowing.jar`

  ```java
  /**
   * This method assumes that the context environment is prepared, or the execution
   * will be a local execution by default.
   */
  public void invokeInteractiveModeForExecution() throws ProgramInvocationException {
    callMainMethod(mainClass, args);
  }
  ```

> Tips：可以在 `flink-conf.yaml` 内配置如下参数远程 debug Client 模块、JobManager 模块和 TaskManager 模块：
env.java.opts.client: "-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5005"
env.java.opts.jobmanager: "-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5006"
env.java.opts.taskmanager: "-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5007"


接下来就是运行 Flink 应用代码，Flink 应用代码结构比较固定，伪代码如下：

```
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
addSource();
addTransformers();
addSink();
env.execute("Flink Example");
```

总结起来就是这么几步：

- 算子关系转换为 StreamGraph，然后再转换为 JobGraph
- 由 StreamExecutionEnvironment 创建一个 PipelineExecutor，这个 executor 提交任务

  `AbstractSessionClusterExecutor#execute` 方法里面核心逻辑如下：

  ```java
  ClusterClient<ClusterID> clusterClient = clusterClientProvider.getClusterClient();
  return clusterClient
      .submitJob(jobGraph)
      .thenApplyAsync(jobID -> (JobClient) new ClusterClientJobClientAdapter<>(
          clusterClientProvider,
          jobID))
      .whenComplete((ignored1, ignored2) -> clusterClient.close());
  ```

  其中 clusterClient 就是 RestClusterClient，可以提交 Restful 请求。
- 最后通过 HTTP 协议向 JobManager 提交任务，代码在 `RestClusterClient#sendRetriableRequest`：

  ```java
  	private <M extends MessageHeaders<R, P, U>, U extends MessageParameters, R extends RequestBody, P extends ResponseBody> CompletableFuture<P>
  	sendRetriableRequest(M messageHeaders, U messageParameters, R request, Collection<FileUpload> filesToUpload, Predicate<Throwable> retryPredicate) {
  		return retry(() -> getWebMonitorBaseUrl().thenCompose(webMonitorBaseUrl -> {
  			try {
  				return restClient.sendRequest(webMonitorBaseUrl.getHost(), webMonitorBaseUrl.getPort(), messageHeaders, messageParameters, request, filesToUpload);
  			} catch (IOException e) {
  				throw new CompletionException(e);
  			}
  		}), retryPredicate);
  	}
  ```

  host 和 port 正是 JobManager，messageHeaders 的具体实例是 `JobSubmitHeaders.getInstance()`，这里指定了其 url

  ![](/img/content/submit-job.png)

## JobManager 运行提交的 Job
JobManager 端接收 HTTP 请求的类是 `DispatcherRestEndpoint`，最底层处理 HTTP 协议是基于 Netty 实现的，底层类是 `LeaderRetrievalHandler`：

```java
/**
 * {@link SimpleChannelInboundHandler} which encapsulates the leader retrieval logic for the
 * REST endpoints.
 *
 * @param <T> type of the leader to retrieve
 */
@ChannelHandler.Sharable
public abstract class LeaderRetrievalHandler<T extends RestfulGateway> extends SimpleChannelInboundHandler<RoutedRequest> {

	@Override
	protected void channelRead0(
		ChannelHandlerContext channelHandlerContext,
		RoutedRequest routedRequest) {

		HttpRequest request = routedRequest.getRequest();
		// other logic
	}

}
```

拿到 HTTP Request 并解析之后，最终会在 `Dispatcher#runJob` 处理：

```java
private CompletableFuture<Void> runJob(JobGraph jobGraph) {
	Preconditions.checkState(!jobManagerRunnerFutures.containsKey(jobGraph.getJobID()));

	final CompletableFuture<JobManagerRunner> jobManagerRunnerFuture = createJobManagerRunner(jobGraph);

	jobManagerRunnerFutures.put(jobGraph.getJobID(), jobManagerRunnerFuture);

	return jobManagerRunnerFuture
		.thenApply(FunctionUtils.uncheckedFunction(this::startJobManagerRunner))
		.thenApply(FunctionUtils.nullFn())
		.whenCompleteAsync(
			(ignored, throwable) -> {
				if (throwable != null) {
					jobManagerRunnerFutures.remove(jobGraph.getJobID());
				}
			},
			getMainThreadExecutor());
}
```

可以看到先创建然后启动 JobManagerRunner，启动之后会将 JobGraph 转化为 ExecutionGraph，基于 ExecutionGraph 开始进行任务调度，任务调度结束就开始正式执行。

## 总结
对照开头的流程图，可以总结 Flink Job 提交并运行的步骤为：
- Client 通过 shell 命令提交任务
- Client 端执行 Flink Job 代码，算子转化为 StreamGraph、StreamGraph 转化为 JobGraph
- Client 通过 HTTP 协议向 JobManager 提交 JobGraph
- JobManager 上的 Dispatcher 组件接收并处理任务提交请求，创建并运行 JobMaster
- JobMaster 向 ResourceManager 申请 slot，若没有之前分配的空闲 slot，ResourceManager 就向 YARN 申请资源
- YARN 分配 container 资源，然后在这个 container 上启动 TaskManager
- TaskManager 启动后，ResourceManager 向其转发 JobMaster 申请 slot 的请求
- TaskManager 向 JobMaster 提供 slot
- JobMaster 拿到 slot 资源后，正式在上面启动 task
