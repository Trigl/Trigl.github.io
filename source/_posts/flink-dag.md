---
layout:     post
title:      "浅谈 Flink DAG：从 DataStream API 到物理执行图"
#subtitle:   "实现 Full Checkpoint"
date:       2022-02-14 01:00:00
author:     "Ink Bai"
catalog:    true
header-style: "text"
#header-img: "/img/archive-bg.jpg"
tags:
    - Flink
---

对于一个数据处理流程，常见的思路就是抽象成一个图，同样的 Flink 中对整个数据处理的抽象也是有向无环图（DAG），并且 Flink 的 DAG 分成了多层，各层之间递进转换，本文将简单探讨下面几个问题：

> **1. Flink DAG 为何要设计成多层？**
> **2. 各层 DAG 都是如何生成的？**

## 从 ~~`HelloWorld`~~ `WordCount` 说起
首先让我们从最简单的 WordCount 代码开始：

```Java
import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.source.datagen.DataGeneratorSource;
import org.apache.flink.streaming.api.functions.source.datagen.RandomGenerator;
import org.apache.flink.util.Collector;

/**
 * Implements a streaming version of the "WordCount" program.
 */
public class StreamingWordCount {

    public static void main(String[] args) throws Exception {
        // get the execution environment
        final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(2);

        // source
        DataStream<String> source = env
                .addSource(new DataGeneratorSource<>(RandomGenerator.stringGenerator(10), 10000, null))
                .setParallelism(1)
                .returns(Types.STRING);

        // parse the data, group it, and aggregate the counts
        DataStream<WordWithCount> counts = source
                .flatMap(new FlatMapFunction<String, WordWithCount>() {
                    @Override
                    public void flatMap(
                            String value, Collector<WordWithCount> out) {
                        for (String word : value.split("\\s")) {
                            out.collect(new WordWithCount(word, 1L));
                        }
                    }
                })
                .keyBy(value -> value.word)
                .sum("count");

        // print the results with a single thread, rather than in parallel
        counts.print();

        env.execute("Streaming WordCount");
    }

    // ------------------------------------------------------------------------

    /**
     * Data type for words with count.
     */
    public static class WordWithCount {

        public String word;
        public long count;

        public WordWithCount() {

        }

        public WordWithCount(String word, long count) {
            this.word = word;
            this.count = count;
        }

        @Override
        public String toString() {
            return word + " : " + count;
        }
    }
}
```

代码很简单我就不介绍了，这段代码对应的 DAG 图如下，包含了各层 DAG 的核心内容，后面我们再基于这张图逐层分析。

<img src="/img/content/flink-dag.jpg" style="max-height:1000px">

## DAG 为何要分层？
分层的目的很简单，就是为了解耦，各层各司其职，更抽象的 Flink Graph 的层次如下图：

![](/img/content/TB1qmtpJVXXXXagXXXXXXXXXXXX.png)

Flink 对图的分层主要基于这样两个维度：

- Stream or Batch？
- 运行阶段：Client/JobManager

第一点分层的考虑点就是批流差异。Flink 顶层分别提供了 Stream API 和 Batch API，由于 Batch 和 Stream 的图结构和优化方法有很大的区别，比如 Batch 有很多执行前的预分析用来优化图的执行，而这种优化并不普适于 Stream，所以第一层分为了 StreamGraph 和 OptimizedPlan，互不影响。

第二点分层的考量是 Flink 任务运行的不同阶段。一个 Flink 任务是从 Client 编译提交，然后在 JobManager 上去调度任务的执行，这两个阶段的不同点在于是否需要关注任务的并行度，Client 只是编译任务，不需要关注并行度；而在 JobManager 要实际调度任务，必然需要关注并行度。

因此在 Client 端又抽象出一层 JobGraph，JobGraph 的责任就是统一 Batch 和 Stream 的图，用来描述清楚一个拓扑图的结构，并且做了 chaining 的优化，chaining 是普适于 Batch 和 Stream 的，所以在这一层做掉。

在 JobManager 端抽象出一层 ExecutionGraph，ExecutionGraph 的责任是方便调度和各个 tasks 状态的监控和跟踪，所以 ExecutionGraph 是并行化的 JobGraph。

而最后一层「物理执行图」，实际上在代码层面并不存在，这里是为了层级统一我们把实际任务在各个节点分配运行的总体情况称为物理执行图。

可以看到，这种解耦方式极大地方便了我们在各个层所做的工作，各个层之间是相互隔离的。

## Stream API
接下来我们从 Flink 任务代码开始，看看各层的 DAG 图都是如何生成的，首先从 Stream API 开始。

以我们最熟悉的 map 操作为例，map 转换将用户自定义的函数 MapFunction 包装到 StreamMap 这个 Operator 中，再将 StreamMap 包装到 OneInputTransformation，它是一个 StreamTransformation。代码结构的层次关系为 `UDF -> StreamOperator -> StreamTransformation`，如下图所示：

![](/img/content/TB12u5yJFXXXXXhaXXXXXXXXXXX.png)

那么最后生成的这个 StreamTransformation 用来干嘛呢？Streaming 任务的环境上下文类 StreamExecutionEnvironment 里面有一个属性：

```Java
protected final List<StreamTransformation<?>> transformations = new ArrayList<>();
```

也就是会把 transformation 保存在 StreamExecutionEnvironment，后续 StreamExecutionEnvironment 就会将这些 transformation 转换为 StreamGraph。注意这里每生成一个 transformation 都有一个对应的 Transformation ID，它是一个从 1 开始递增的静态变量。

总结起来，Stream API 的作用非常简单，就是将用户的 UDF 函数转换成 transformation，然后保存到 StreamExecutionEnvironment。

## StreamGraph
StreamGraph 是由用户代码转换而来的第一层 DAG 图，只是简单地根据代码逻辑创建出对应的顶点和边，比较简单。

`StreamExecutionEnvironment#execute` 会触发 StreamGraph 和 JobGraph 的生成，我们对照文章开头的图中截取出 StreamGraph 的部分来了解生成过程：

![](/img/content/streamgraph.jpg)

StreamGraph 最核心的数据结构包括：

- StreamNode：由 Transformation 转换而来，里面存储了用户的 UDF 对象、输入输出的序列化方法、以及对应的 Transformation ID 等信息。
- StreamEdge：用来连接上下游 StreamNode，并且存储分区信息 StreamPartitioner

生成 StreamGraph 的代码在 `StreamGraphGenerator#generate`，具体源码自行研究，这里介绍下主要步骤：

- 遍历 StreamGraphGenerator 中存储的 transformation 集合
- 从 Source Transformation 开始，将 Transformation 转化为 StreamNode，并生成 StreamEdge 连接上游和下游的 StreamNode

## JobGraph
事实上 JobGraph 的存在才是真正为了后续 JobManager 调度任务做准备，JobGraph 图中的每一个顶点需要对应到 JobManager 中调度的一个单位，基于此，JobGraph 相对于 StreamGraph 最大的不同就是生成 OperationID 用于任务调度恢复、同时 chain 算子，从而减少数据在不同节点之间流动所产生的序列化、反序列化、网络传输的开销。

还是先从文章开头图中截取出 StreamGraph -> JobGraph 转化的部分：

![](/img/content/jobgraph.jpg)

JobGraph 的核心数据结构：

- JobVertex：JobGraph 的顶点，跟 StreamNode 的区别是，它是 Operator Chain 之后的顶点，会包含多个 StreamNode
- IntermediateDataSet：由一个 Operator（可能是 source，也可能是某个中间算子）产生的一个中间数据集，我们可以把 JobEdge 看作是 IntermediateDataSet 的消费者，那么 JobVertex 自然就是生产者了。
- JobEdge：JobGraph 中的边，连接一个 IntermediateDataSet 跟一个要消费的 JobVertex，也会存 DistributionPattern 类型，而 DistributionPattern 跟 Partitioner 有关。如果是 ForwardPartitioner 或者 RescalePartitioner，DistributionPattern 就是 POINTWISE 模式；如果是其他 Partitioner，DistributionPattern 就是 ALL_TO_ALL 模式。
- StreamConfig：图中没有这个数据结构，但是比较重要，每一个 StreamNode 都会创建一个对应的 StreamConfig。StreamConfig 中保存了这个算子（operator） 在运行是需要的所有配置信息，可序列化，会发送给 JobManager 和 TaskManager。最后在 TaskManager 中起 Task 时，需要从这里面反序列化出所需要的配置信息，其中就包括了含有用户代码的 StreamOperator。

构造 JobGraph 的代码在 `StreamingJobGraphGenerator#createJobGraph`，这里还是仅介绍主要步骤：

- 广度优先遍历 StreamGraph 并且为每个 SteamNode 生成唯一的 hash ID
- 从所有 source 节点开始遍历，递归调用 `createChain` 方法构建 chain，具体实现类似于贪心算法
  - 找出当前 StreamNode 的所有出边，然后遍历这些出边，看出边上下游算子能否 chaining 在一起
  - 能够 chaining 在一起的话，就继续向下游节点递归尝试 chain 更多；不能 chaining 在一起的，结束从下一个节点重新开始。
  - chain 结束之后，对于 chain 起始节点，需要创建 JobVertex。对于 chain 子节点，不需要创建自己的 JobVertex，仅将 StreamConfig 添加到该 chain 的 config 集合。
  - 最后一步是将 JobVertex 连接起来，连接的方式是当前节点主动连接上游节点，首先创建上游节点对应的 IntermediateDataSet，然后再创建 JobEdge 把上游节点的 IntermediateDataSet 和当前节点的 JobVertex 连接起来，连接关系如下图，可以看到一个 JobVertex 会包含多个 JobEdge 和 IntermediateDataSet。
  ![](/img/content/jobgraph-connect.png)

> **关于唯一 hash ID**
相同任务在恢复的时候各个算子的 ID 需要保证不变，这样才能基于这个 ID 从 Checkpoint 中拿到自己的状态信息进行恢复。StreamGraph 内的 StreamNode 的 ID 从 Transformation ID 转换而来，而 Transformation ID 是一个不断递增的静态变量，每次任务重启的时候会发生变化，因此我们需要引入另一套 ID 机制去标识作业，这就是 Operator ID，也就 JobGraph 对 StreamNode 生成的唯一 hash ID。<br>
这个 ID 生成算法取决于下面几个因素：
- 该算子所处的位置（下标）
- 该算子 chain 的下游算子数
- 上游算子（hash 结果还要与上游算子的哈希值进行异或操作）

> **如何判断上下游算子是否可以 chaining**
>1. 下游仅有上游一个输入
>2. 上游和下游在同一个 slotSharingGroup（默认满足）
>3. 上游节点和下游节点的 ChainingStrategy 必须符合一定条件，ChainingStrategy 有三种：ALWAYS、NEVER、HEAD
>4. 上游和下游必须通过 ForwardPartitioner 发送数据
>5. shuffle 模式不能是 Batch
>6. 上下游算子的并行度相同
>7. StreamGraph 的 chaining 配置项为 true

## ExecutionGraph
JobMaster 会根据 JobGraph 生成对应的 ExecutionGraph，ExecutionGraph 是 Flink 作业调度时使用到的核心数据结构，它包含每一个并行的 task、每一个 intermediate stream 以及它们之间的关系。

还是先上图 JobGraph -> ExecutionGraph 转化的部分：

![](/img/content/executiongraph.jpg)

ExecutionGraph 的核心数据结构：

- ExecutionJobVertex：与 JobGraph 中的 JobVertex 对等
- ExecutionVertex：是 ExecutionJobVertex 并行执行的子任务，对应后续实际运行的 Task，每个 ExecutionJobVertex 内 ExecutionJobVertex 的个数就是算子的并行度
- Execution：对 ExecutionVertex 的一次实际执行，每次执行都会生成新的 Execution，由 ExecutionAttemptId 来唯一标识
- IntermediateResult：与 JobGraph 中的 IntermediateDataSet 对等
- IntermediateResultPartition：可以看做是 IntermediateResult 并行化下的一个子结构，用来表示上面 ExecutionVertex 的一个输出分区
- ExecutionEdge：将 ExecutionVertex 和 IntermediateResultPartition 连接起来

核心代码在 `ExecutionGraphBuilder#buildGraph` 中，主要步骤为：

- 先对 JobGraph 中的 JobVertex 按拓扑结构做排序，所谓拓扑排序，即保证如果存在 A -> B 的有向边，那么在排序后的列表中 A 节点一定在 B 节点之前。
- 根据排序后的 source JobVertex 集合构建 ExecutionGraph，基本思路就是先创建与 JobGraph 数据结构对应的数据结构，如 ExecutionJobVertex、IntermediateResult，然后再创建这些数据结构对应的并行化版本下的数据结构，如 ExecutionVertex、IntermediateResultPartition、Execution。
- 最后创建 ExecutionEdge 把 ExecutionVertex 与 IntermediateResultPartition 连接起来，这里的连接取决于前面创建 JobGraph 过程中新建 JobEdge 时指定了什么 DistributionPattern，如果是 POINTWISE 模式，此时 ExecutionVertex 只会与部分的 IntermediateResultPartition 连接起来；如果是 ALL_TO_ALL 模式，这个 ExecutionVertex 会与 IntermediateResult 对应的所有 IntermediateResultPartition 连接起来。

## 物理执行图
ExecutionGraph 是在创建 JobMaster 时就构建完成的，之后就可以被调度执行了。调度的实现在 `DefaultScheduler#startSchedulingInternal` 内，主要步骤为：

- 按照拓扑顺序为所有的 ExecutionJobVertex 分配资源，其中每一个 ExecutionVertex 都需要分配一个 slot
- 所有的节点资源都获取成功后，会逐一部署 Execution
- 使用 TaskDeploymentDescriptor 来描述 Execution，并提交到分配给该 Execution 的 slot 对应的 TaskManager，最终被分配给对应的 TaskExecutor 执行

再来看一下 ExecutionGraph -> 物理执行图的转换：

![](/img/content/physicalgraph.jpg)

这个物理执行图实际上是 TaskManager 内 task 的执行情况，从中我们可以看到 Task 的关键数据结构：

- Task：Execution 被调度后在分配的 TaskManager 中启动对应的 Task，Task 包裹了具有用户执行逻辑的 operator。
- ResultPartitioner：可以看做是 Task 数据的发送缓冲层，而 Task 是 ResultPartitioner 的生产者。
- ResultSubpartition：是 ResultPartition 的一个子分区，其其数目要由下游消费 Task 数和 DistributionPattern 来决定。记录会基于不同的 StreamPartitioner 选择不同的子分区分发，例如 ForwardPartitioner 会固定选择首个子分区，而 RebalancePartitioner 会轮询选择所有子分区分发。
- InputGate：可以看做是 Task 的接收缓冲层，也可以把它看做是 ResultPartitioner 的消费者，每个 InputGate 消费了一个或多个的 ResultPartition。
- InputChannel：每个 InputGate 会包含一个以上的 InputChannel，用来实际消费子分区 ResultSubpartition 的数据，与其一一相连。

## 总结
本文介绍了一个 Flink 作业，从 API 层面，到 Client 端提交，到 JobManager 调度运行起来，其 Graph 的转换会经过下面几个阶段：

- Steam API：将用户编写的函数转化为多个 Transformation，用于后续构建 DAG
- StreamGraph：根据编写的代码生成最初的 Graph，它表示最初的拓扑结构
- JobGraph：这里会对前面生成的 Graph 做一些优化操作（如 operator chain 等），最后会提交给 JobManager
- ExecutionGraph：JobManager 根据 JobGraph 生成 ExecutionGraph，是 Flink 调度时依赖的核心数据结构
- 物理执行图：JobManager 根据生成的 ExecutionGraph 对 Job 进行调度后，在各个 TM 上部署 Task 后形成的一张虚拟图。

## Refer
[Flink 原理与实现：架构和拓扑概览](http://wuchong.me/blog/2016/05/03/flink-internals-overview/)
[Flink 原理与实现：如何生成 StreamGraph](http://wuchong.me/blog/2016/05/04/flink-internal-how-to-build-streamgraph/)
[Flink 原理与实现：如何生成 JobGraph](http://wuchong.me/blog/2016/05/10/flink-internals-how-to-build-jobgraph/)
[【Flink】源码笔记—StreamGraph 到 JobGraph](https://segmentfault.com/a/1190000040903795)
[Flink Streaming 作业如何转化为 JobGraph](https://matt33.com/2019/12/09/flink-job-graph-3/)
[Flink 如何生成 ExecutionGraph](https://matt33.com/2019/12/20/flink-execution-graph-4/)
[Flink 源码阅读笔记（3）- ExecutionGraph 的生成](https://blog.jrwang.me/2019/flink-source-code-executiongraph/)
