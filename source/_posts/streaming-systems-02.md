---
layout:     post
title:      "「Notes」Streaming Systems 第二章"
subtitle:   "大规模数据处理的四个核心问题：What, Where, When And How"
date:       2022-02-19
author:     "Ink Bai"
catalog:    true
# header-style: "text"
header-img: "/img/archive-bg.jpg"
tags:
    - Streaming Systems
---

本章可以说是干货满满，作者从宏观上讲了大规模数据处理需要解决的四个核心问题：

1. What：处理的数据最终要计算出什么结果？
2. Where：在什么地方计算出结果？
3. When：何时最终输出结果？
4. How：如何优化结果？

那么这四个问题的答案分别是什么，我们接着看。

## 1、What：Transformation
Transformation 指的是对数据进行转换，所以第一个问题「处理的数据最终要计算出什么结果」，答案就是最终得到的是数据的转换（可能多次转换）。

首先让我们看一组示例数据，这组数据展示了队伍中各个成员的得分，将作为后续所有代码实例的原始数据：

![](/img/content/sample.png)

我们用 Beam 代码来解释 Transformation 的含义，首先明确下面两个 Beam 中的基本变量：

- `PCollections`：代表可以进行并行转换的大数据集，开头的 P 就代表可以并行执行。
- `PTransforms`：用于应用到 PCollections 然后再产生新的 PCollections，可以有以下几种 PTransforms，一对一转换、多对一的聚合转换，以及多个 PTransforms 的组合转换。

![](/img/content/stsy_0202.jpg)

Talk is cheap, show me the code. 让我们看一段 Beam 中转换的代码，这段代码用于计算每个 team 各自的总成绩：

```java
PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals = input.apply(Sum.integersPerKey());
```

首先通过 IO 操作读取数据源，数据源的数据就是上面表格中的示例数据，然后应用一个一对一转换 `ParseFn` 对数据源做解析，产生一个新的 `PCollection`，它的类型是 KV 对。之后再进行一次转换，应用 `Sum.integersPerKey()` 函数，即对相同 key 对应的 value 累加，最后仍然产生一个新的 `PCollection`，类型还是 KV 对，这个就是我们要的最终数据结果。

下图展示了累加分数的进程：

![](/img/content/stsy_0203.jpg)

为了方便展示，这里仅展示了单独一个 team 的转换过程，实际上在真实数据管道中，相似的操作和相似的 team 可以同时在多台机器上运行。

通过上面的代码示例，我们明白了，Transformation，也就是转换，就是一个个的函数，可以自由的把数据类型 A 转换成 B，直到得到我们想要的结果。

这个例子进行的是批处理，对整个数据集进行的转换，然而这种模式并不能应用在流处理上，因为无限流数据理论上没有终点，没办法等所有数据都输入完成。此时就引出了第二个问题，数据应该在在什么地方进行转换呢，答案就是在窗口内，下面让我们简单复习一下窗口。

## 2、Where：Windowing
从第一章我们知道，窗口具有以下三种类型：

![](/img/content/stsy_0108.png)

现在还是用代码实例来看一下窗口的用法，仍然是上面计算队伍得分的例子，不同之处是这里我们在窗口内进行输出：

```java
PCollection<KV<Team, Integer>> totals = input
  .apply(Window.into(FixedWindows.of(TWO_MINUTES)))
  .apply(Sum.integersPerKey());
```

下图展示了单独某个 key 在窗口内计算的过程：

![](/img/content/stsy_0205.jpg)

可以看到不同于之前的全局计算，这里分成了 4 个时间窗口，每个窗口大小是两分钟。

## 3、When：Trigger + Watermark
上面解决了在哪里的问题，现在就要解决何时的问题了，此时需要引入新的概念 trigger 和 watermark。

#### 3.1、Trigger
正如上面所说，trigger 用来回答这个问题，「在处理时间中，什么时候输出结果」，每次 trigger 都会输出一个结果，而每一个输出称为 pane（窗格）。

Trigger 通常有两种类型：

- 变更性 trigger 
- 完整性 trigger 

**3.1.1、变更性 trigger**
一个窗口内可以进行多次 trigger，每次 trigger 产生一个变化了的 pane，可以是每来一条记录变更一次 pane 或者是周期性变更如一分钟变更一次。变更周期的选择需要综合考虑延迟和成本。

下面我们先看一下这种类型的实例，首先看每来一条记录都进行一次 trigger：

```java
PCollection<KV<Team, Integer>> totals = input
  .apply(Window.into(FixedWindows.of(TWO_MINUTES))
               .triggering(Repeatedly(AfterCount(1))))
  .apply(Sum.integersPerKey());
```

![](/img/content/stsy_0206.jpg)

从上图可以看到单个窗口内每来一条记录就会产生一次输出，也就是产生一个 pane。这种变更性的 trigger 特别适合数据写到表里的场景，表的数据会实时更新，并且随着时间推进结果也愈发精确。

但是每来一条记录都 trigger 的缺点就是成本太高，会消耗大量资源，特别是 key 数量很多的时候，此时就可以考虑周期性变更 trigger，每个周期延迟固定时间后开始下一周期，如每秒钟甚至每分钟变更一次。可以依据是否对齐延迟细分为两种类型：

1.对齐延迟（aligned delay）：严格按照处理时间切分成相同的时间周期，例子如下：

```java
PCollection<KV<Team, Integer>> totals = input
  .apply(Window.into(FixedWindows.of(TWO_MINUTES))
               .triggering(Repeatedly(AlignedDelay(TWO_MINUTES)))
  .apply(Sum.integersPerKey());
```

![](/img/content/stsy_0207.jpg)

上图就是按照实际处理时间切分 pane，Spark Streaming 的微批思想就类似于这种对齐 trigger，这种 trigger 的缺点就是所有的结果都是在同一时间输出，就会导致周期性地负载变高，此时就可以选择非对齐延迟的 trigger。

2.非对齐延迟（unaligned delay）：窗口内对应的 key 实际到达数据后才开始计算延迟的时间周期

```java
PCollection<KV<Team, Integer>> totals = input
  .apply(Window.into(FixedWindows.of(TWO_MINUTES))
               .triggering(Repeatedly(UnalignedDelay(TWO_MINUTES))
  .apply(Sum.integersPerKey());
```

![](/img/content/stsy_0208.jpg)

对比上面对齐延迟的图，可以很明显看出非对齐延迟可以打散负载，对于大规模数据处理来说是一种更好的选择。

**3.1.2、完整性 trigger**
讲完了变更性 trigger，再来看完整性 trigger。完整性 trigger 是指这个窗口的数据全部到达之后才会输出结果，这类似于批处理，只不过是数据限定在一个窗口期内。

那么现在新的问题来了，如何能确定数据数据全部到达了，也就是如何确保数据的「完整性」呢，此时就需要 watermark 登场了。

#### 3.2、Watermark
不同于上面 trigger 是解决 When 问题的直接答案，Watermark 是解决 When 问题的一个必要条件，那就是标志数据在事件时间维度下的完整性。直接解释就是「当 watermark 称为事件时间下的某个值的时候，就标志着这个事件事件之前的数据已经全部到齐」。

**3.2.1、理解 watermark**
先回忆一下在第一章中处理时间和事件时间之间的进度关系图：

![](/img/content/stsy_0209.jpg)

中间曲折的线就代表了 watermark，展示了随着处理时间的推进，事件时间的最新进度。
从这张图来抽象，其实本质上 watermark 就是一个处理时间到事件时间的函数，F(P)->E，但是注意函数输出是数据流事件时间的最新进度，最新进度指的不是当前最大的事件时间，而是最小的事件时间，保证之后到来的所有记录的事件时间都应当比这个进度的时间大。
那么如果出现了比这个小的记录怎么办？这些数据就是迟到的数据，后续我们会说明如何处理这种情况。

**3.2.2、watermark 的问题**
明确了 watermark 的定义，我们就可以实现 `3.1.2、完整性 trigger` 了。现在我们就基于 watermark 来看一个实现完整性 trigger 的代码实例：

```java
PCollection<KV<Team, Integer>> totals = input
  .apply(Window.into(FixedWindows.of(TWO_MINUTES))
               .triggering(AfterWatermark()))
  .apply(Sum.integersPerKey());
```

> 完整性 trigger：完美型 watermark 和启发型 watermar

![](/img/content/stsy_0210.jpg)

图中左右两边分别展现了两种实现的 watermark：

- 完美型 watermark：处理时间到事件时间的映射完全准确，能够完全保证数据的完整性，不会有任何数据迟到的情况。
- 启发型 watermark：一般而言，完美只存在于想象当中，在大多数分布式场景中，我们能做的是尽可能根据进入的记录的各种信息来预估此时的 watermark，例如网络一般延迟多久、数据是否有序等。大多数情况下，这种预测是比较准确的，但是也会有不准的情况，此时就会导致有些数据被认为是迟到的数据。

我们可以发现 watermark 或者说完整性这个概念本身，可能会存在下面两个问题：

- 数据延迟：用上面图左完美 watermark 来举例，由于 [12:00, 12:02) 时间内的一条数据晚到了很久，这个窗口在很久之后才被 trigger，这就引发了连锁反应，它后面的几个窗口 trigger 时间都推迟了。
- 数据不完整：用上面图右启发型 watermark 来举例，它预测的 watermark 比较早，数据不会延迟很久，但是却导致有一条数据迟到了没有包含在这个窗口期内，也就是造成了数据不完整。而某些场景是很关心数据的完整性的，典型的例子就是 outer join，如果没有 watermark 这样的概念，你怎么知道什么时候两条流应该停止 join 输出结果呢？

所谓鱼与熊掌不可兼得，不能又想要数据完整又想要数据不延迟，能不能「我都要」呢？想一想前面 `3.1.1、变更性 trigger` 中的优点，可以低延迟地更新数据的结果，每次变更数据都更准确，如果变更性 trigger 和 watermark 所提供的完整性结合起来会怎样呢？

#### 3.3、前置/准时/后置 trigger
如何把变更性 trigger 和 watermark（完整性 trigger）相结合呢？

- 准时 trigger：首先要保证 watermark 变成窗口结尾的时间时得有一个 trigger 输出结果，换句话说，就是得先有一个完整性 trigger 保证数据准确。
- 前置 trigger：在这个窗口最终完整 trigger 之前，使用变更性 trigger 快速更新数据，保证数据低延迟。
- 后置 trigger：在这个窗口最终完整 trigger 之后，再使用另一个变更性 trigger 来处理迟到的数据，保证 window 结果准确。注意后置 trigger 可以设置每来一条记录就输出一次结果，是因为数据迟到的场景并不是常态，成本不会太大。

让我们看下代码实现：

```java
PCollection<KV<Team, Integer>> totals = input
  .apply(Window.into(FixedWindows.of(TWO_MINUTES))
               .triggering(
                AfterWatermark()
                  .withEarlyFirings(AlignedDelay(ONE_MINUTE))
                  .withLateFirings(AfterCount(1))));
```

> 前置/准时/后置 trigger：完美型 watermark 和启发型 watermar

![](/img/content/stsy_0211.jpg)

将本图和上面 `完整性 trigger：完美型 watermark 和启发型 watermar` 对比，你会发现不论是完美的 watermark 还是启发型的 watermark，他们貌似差距不大了，都能在数据准确的基础上还能保持快速更新输出结果。

一切看起来都很完美，但是还有一个问题，就是当你设置了一个后置 trigger，也就意味着即使某个窗口在完整性 trigger 输出结果之后，还是需要保存其状态，以方便后续到来的数据更新这个结果，那么这个状态需要保存多久呢？

#### 3.4、允许数据延迟
一般来说，没必要保存特别长时间的窗口，因为除了可能耗尽磁盘空间以外、过去太长时间的数据往往没有什么价值了，继续保存也没有意义。
因此，真实场景中对无序数据处理的时候，往往需要对其处理的窗口的生命周期做出限制，即需要定义一个具体的延迟时间阈值，这个阈值定义了一条记录超过 watermark 多久之后到达就不会再被处理了，也就是直接丢掉，同时也意味着超过这个时间的窗口也会被删除。这是合理的。

同时注意，这里定义的延迟时间，也是指的 watermark，比如有一个窗口在 [12:00, 12:02) 区间，我们定义延迟时间阈值是 1 分钟，那么此时我们会保留该窗口的状态，一直等到 watermark 到达了 12:03，才会彻底删除该窗口。

这里为什么不用处理时间来简单定义一个延迟阈值呢，比如在 watermark 过了窗口末尾时间之后，定义处理时间下的 10 分钟为延迟时间阈值，这样也可以保证固定 10 分钟以后删除窗口。这样虽然简单但是存在一个问题，如果此时系统 crash 了几分钟，当重新恢复的时候已经过了 10 分钟，这就导致 window 无法处理它本可以处理的某些迟来的数据。而基于 watermark 的延迟真实反映了数据流本身的时间延迟情况，外部系统不会影响到数据流的逻辑。

那么现在再来看以下加上延迟时间阈值的代码实例：

```java
PCollection<KV<Team, Integer>> totals = input
  .apply(Window.into(FixedWindows.of(TWO_MINUTES))
               .triggering(
                 AfterWatermark()
                   .withEarlyFirings(AlignedDelay(ONE_MINUTE))
                   .withLateFirings(AfterCount(1)))
               .withAllowedLateness(ONE_MINUTE))
 .apply(Sum.integersPerKey());
```

![](/img/content/stsy_0212.jpg)

不过最后再强调一点，凡事无绝对，具体情况还是要具体分析。虽然大部分系统有必要设置一个延迟时间阈值，但是如果你的系统的确需要结果非常精确，比如计算网站每天的 PV，同时 key 的数量不大，比如只有有限几个网站，那完全可以把窗口的生命周期设置为无限。

## 4、How：Accumulation
上文讲到 trigger 分为可以按照设置的时间分为前置 trigger、准时 trigger、后置 trigger，这些 trigger 在不同时间会为同一个窗口更新优化结果，使结果越来越准确。那么问题来了，「结果如何进行优化呢」，答案就是通过「Accumulation（聚合）」的方式。

前面例子中多个 trigger 输出的结果进行优化的方式是将新的 pane 输出的结果和旧的 pane 的结果累加起来，这种聚合方式称为「累加」。事实上，总共有三种聚合方式：

- Discarding（丢弃）：指每次输出一个 pane 之后，其结果就被丢弃，也就意味着不同的 pane 之间的结果互不影响相互独立。
- Accumulating（累加）：每次输出一个 pane 之后，保存结果，下次产生新的 pane 之后与之前保存的结果累加，然后再覆盖保存的结果。比如可以通过一个 k/v 存储的外部系统如 HBase 来存储结果。
- Accumulating and retracting（累加并撤销）：这种模式类似于累加模式，不同之处在于当产生一个新 pane 之后，除了累加，还会对前一个 pane 做出额外的撤销操作。这种模式本质上是在说「我上次告诉你结果是 X，现在我发现我错了。去掉我上次告诉你的 X，换成新的结果 Y」。

比如前面动图 `前置/准时/后置 trigger：完美型 watermark 和启发型 watermar` 在 [12:06, 12:08) 区间范围内有两个 pane，下面这个表格展示了这三种聚合方式对应的最终结果：

![](/img/content/accumulation.jpg)

代码实现如下：

```java
PCollection<KV<Team, Integer>> totals = input
  .apply(Window.into(FixedWindows.of(TWO_MINUTES))
               .triggering(
                 AfterWatermark()
                   .withEarlyFirings(AlignedDelay(ONE_MINUTE))
                   .withLateFirings(AtCount(1)))
               .discardingFiredPanes())
               //.accumulatinFiredPanes())
               //.accumulatingAndRetractingFiredPanes())
  .apply(Sum.integersPerKey());
```

最后可以想象到，这三种模式需要的存储资源和计算成本也是逐渐增加的，选择何种优化模式其实就是在数据准确性、延迟和成本间做一个平衡。

## 5、总结
第一章和第二章的内容是 streaming system 的基础，通过这两章内容，我们首先知道了流系统的一些基本概念：

- 事件时间 vs. 处理时间：记录发生时间和在系统中处理的时间。
- 窗口：处理无界数据流的常用工具，将无界数据流切成一块块临时的有界数据。
- Trigger：用于确定何时输出具体的结果。
- Watermark：表示事件时间进度的概念，为处理无序的、无界数据的完整性(以及丢失的数据)提供了一种推理方法。
- Accumulation：随着时间的推移，对单个窗口的结果聚合优化。

现在我们就可以简答开头四个问题的答案了：

1. What：处理的数据最终要计算出什么结果？= 转化
2. Where：在什么地方计算出结果？= 窗口
3. When：何时最终输出结果？= trigger+watermark
4. How：如何优化结果？= 聚合
