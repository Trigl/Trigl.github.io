---
layout:     post
title:      "如何学习开源项目？"
date:       2019-05-10
author:     "Ink Bai"
catalog:    true
header-img: "/img/post/learn-opensource-project.jpg"
tags:
    - 方法论
---

> Winter is coming.
对于工程师来说，度过寒冬的方法一个是抱团取暖，所以才有 996.icu。但更重要的是提升自己的硬实力，而学习开源项目，就是自我提升的一大利器。

## 没有开源的上古时代
幻想一下上古时代程序员小菜，想要实现个东西，找了半天发现好像没有人做过。

哦，可能也没地方可找-。-

那就只能自己做喽，吭呲吭呲大半年，上线收工。

想的美，工作才刚刚开始！

小菜，客户又遇到问题了！

小菜，你这软件速度太慢了！

小菜，怎么我用着用着软件又闪退了，你特么用屁股写的代码吧！

正当小菜濒临崩溃的时候，这个时候发现隔壁公司小牛做了个类似的东西，速度杠杠的，界面还好看，于是小菜激动的问小牛是怎么做到的。

然而小牛直接告诉他，商业机密，无可奉告。

我想如果这个时候有一个类似的开源项目放在小菜面前，他一定会做梦都笑醒的。
## 我为什么学习开源项目？
到了今天，使用开源项目就像使用水电煤一样普遍和简单，然鹅人是一种奇怪的生物，越是容易得到的东西越不懂的珍惜，反正我最初对待开源项目就是这样。

最开始使用开源项目，就感觉它们跟电脑手机一样，只是个工具而已。工具嘛，用来解决问题就好了，研究研究？没兴趣。有那时间百度了一百个答案了。

然后准备跳槽了，开始面试：

面试官甲：看过 HashMap 的源码吗？

面试官乙：如何实现一个线程池？（看我不甚理解的样子）换句话说，讲一下 `ThreadPoolExecutor` 的底层实现。

面试官丙：Netty 的性能为什么这么高？

我的天呐，这些面试官都是变态吗？怎么都问这些开源项目的底层实现。咋的，我特喵的买个手机，不只得会用还得会造啊。

算了，看在辣个男人的份上，我学！哎你别说，稍微看了点核心源码，（就是面试重点嘎嘎），逐渐可以照猫画虎，写的 bug 好像少了一点了，而且定位 bug 的能力越来越强了呦，我好像不再是重启型程序员了呢！

再接着往后呢，开始负责一些系统设计的工作，这个时候发现，可以从开源项目中借鉴（抄袭）很多架构设计思想，然后慢慢的也就变成自己的了。

总结下来，我学习开源项目的过程是这样的：

- 面试要考，所以我要学习
- 这个 bug 网上没有解决方法啊，看看源码吧
- 卧槽还可以这样写代码，好巧妙啊，学到了学到了这波不亏
- 这个部分怎么设计呢？先看看 Spark 是怎么做的吧

可以看到一开始听到学习开源项目，我是拒绝的，后面开始看也是出于很功力的目的。但事情往往就是这样，并不都是你热爱它然后才去做，而是你去做了之后才发现它其实蛮有趣的。
## 学习开源项目的原则
在我们正式准备开始学习某个开源项目之前，首先明确几个基本点。

> 原则一：学习开源项目 != 看源码

学习开源项目的方式是采取自顶向下的学习方法，源码不是第一步，而是最后一步。

不要一上来就去看源码，而是要基本掌握了功能、原理、关键设计之后再去看源码，看源码的主要目的是为了学习其代码的写作方式，以及关键技术的实现。

例如，Redis 的 RDB 持久化模式“会将当前内存中的数据库快照保存到磁盘文件中”，那这里所谓的“数据库快照”到底是怎么做的呢？在 Linux 平台上其实就是 fork 一个子进程来保存就可以了；那为何 fork 子进程就生成了数据库快照了呢？这又和 Linux 的父子进程机制以及 copy-on-write 技术相关了。

通过这种方式，既能够快速掌握系统设计的关键点（Redis 的 RDB 模式），又能够掌握具体的编程技巧（内存快照）。

到底看不看源码，要根据具体情况而定。

如果你的目的是学习某种语言，那我强烈建议你仔细阅读开源项目的源码，你可以在其中学习到优秀的编程实践，如何设计代码结构，如何抽象，使用了哪些设计模式，如何使用这种语言构建一个完整的项目等等等等。

如果你是一个沉浸开源多年的老司机，正在做技术调研或者架构设计，通过官方文档和你的知识经验积累已经足够了解某开源项目的底层实现，那么看不看源码就没那么重要了。

> 原则二：带着问题去学习

学习开源项目之前，首先弄明白它为了解决什么问题，然后自己想一下，如果让你来解决这个问题你会怎么办。

同样的，看到一些关键的技术点，也可以想一下自己如何解决这个问题。只有带着这样的疑问，你才能更好理解作者的设计思想和理念，理解这个项目可以解决的痛点，理解地才能更深刻。

> 原则三：Read is cheap, run the code

不管是看官方文档还是阅读源码，一定要动手去做，去试，让代码跑起来。

你可以自己写 Demo，也可以亲自搭建源码阅读环境 debug 源码，切勿只看不做眼高手低。

> 原则四：不要过于关注细节

俗话说细节决定成功，其实也得看是什么样的细节。

对于一个程序员来说，工作中对每个变量的命名都谨慎小心和每天早上把自己的头发梳地一丝不苟，这两个细节的重要程度绝对是不同的。

对于开源项目也是一样的，我们要抓住主要矛盾，暂时放过次要矛盾。

对于系统的核心原理和思想，一定要拿出咬定青山不放松的精神，死磕到底不离不弃。

对于一些类似边缘条件判断、参数检测的细枝末节，就让它们自己玩去吧。

那么如何判断哪些是关键细节哪些是无关细节呢？很简单，对应到代码，直接看命名，好的开源代码通过命名就知道重不重要。
## 学习开源项目的步骤
下面介绍如何安装“自顶而下”的步骤来学习：
#### 第一步：基础性了解学习
目标是达到基础学习的层次，对项目有<u>**大概性的了解**</u>，包括项目背景，解决的问题场景，项目功能，使用场景，基本的API使用。通过查找官方文档、相关博客、视频资料学习即可。

通过对系统有大概性了解之后，会自然而然有一些疑问，例如实现的原理，优缺点等，后续学习带着这些疑问进行学习会更高效

#### 第二步：系统性学习与实践
目标是达到检视学习的层次，对项目有<u>**系统性、全面性的了解**</u>，包括项目的功能、组成模块、基本原理、使用场景、配置项、API使用、与其他类似项目的优缺点比较等。

方法步骤如下：

<u>**安装运行**</u>
按照相关文档，安装运行项目。在这个过程中，需要关注：

- 系统的依赖组件：因为依赖组件是系统设计和实现的基础，可以了解系统一下关键信息，例如 Memcached最重要的依赖是高性能的网络库 libevent，我们就能大概推测 Memcached 的网络实现应该是 Reactor 模型的。
- 安装目录：常见的安装目录是conf存放配置文件，logs存放日志文件，bin存放日志文件，而不同项目有些特殊目录，比如Nginx有html目录，这种目录能促使我们带着相关疑问继续去研究学习，带着问题去学习效率是最高的。
- 系统提供的工具：需要特别关注命令行和配置文件，它们提供2个非常重要的关键信息，系统具备哪些能力和系统将会如何运行。这些信息是我们学习系统内部机制和原理的一个观察窗口。通常情况下，如果对每个命令行参数和配置项的作用和原理基本掌握了解的话，基本上对系统已经很熟悉了。实践中，可以不断尝试去修改配置项，然后观察系统有什么变化。

<u>**系统性研究原理与特性**</u>
这点相当重要，因为只有清楚掌握技术的原理特性，才能算真正掌握这门技术，才能做架构设计的时候做出合理的选择，在这个过程中，需要重点关注：

- 关键特性的基本实现原理：关键特性是该开源开源项目流行的重要卖点，常见的有高性能、高可用、可扩展等特性，项目是如何做到的，这是我们需要重点关注的地方。
- 优缺点比对分析：优缺点主要通过对比来分析，即：我们将两个类似的系统进行对比，看看它们的实现差异，以及不同的实现优缺点都是什么。典型的对比有 Memcached 和 Redis、Kafka和ActiveMQ、RocketMQ的比较。
- 使用场景：项目在哪些场景适用，哪些场景不适用，业界适用常见案例等。

在此阶段可以通过学习官方技术设计文档文档，架构图，原理图，或者相关技术博客，通常比较热门的开源项目都有很多分析文档，我们可以站在前人的基础上避免重复投入。但需要注意的是，由于经验、水平、关注点、使用的版本不同等差异，不同的人分析的结论可能有差异，甚至有的是错误的，因此不能完全参照。一个比较好的方式就是多方对照，也就是说看很多篇分析文档，比较它们的内容共同点和差异点。

同时，如果有些技术点难以查到资料，自己又不确定，可以通过写Example进行验证，通过日志打印、调试、监测工具观察理解具体的细节。例如可以写一个简单程序使用Netty，通过抓包工具观察网络包来理解其中的实现。

#### 第三步：系统测试
如果是只是自己学习和研究，可以参考网上测试和分析的文档，但是如果要在生产环境投入使用必须进行测试。因为网上搜的测试结果，不一定与自己的业务场景很契合，如果简单参考别人的测试结果，很可能会得出错误的结论，或者使用的版本不同，测试结果差异也比较大。

要特别注意的是，测试必须建立在对这个开源项目有系统性了解的基础上，不能安装完就立马测试，否则可能会因为配置项不对，使用方法不当，导致没有根据业务的特点搭建正确的环境、没有设计合理的测试用例，从而使得最终的测试结果得出了错误结论，误导了设计决策。

下面提供测试常见的思路参考，需要根据具体项目具体业务进行测试用例的设计。

- 核对每个配置项的作用和影响，识别出关键配置项
- 进行多种场景的性能测试
- 进行压力测试，连续跑几天，观察 CPU、内存、磁盘 IO等指标波动
- 进行故障测试：kill，断电、拔网线、重启 100 次以上、倒换等

#### 第四步：关键源码学习
<u>**看源码有一条捷径，就是从最初的有核心代码的版本，一般是 1.0 版本的 Release，往后每一次大的 Release 我们都需要了解一下。**</u>

钻研、领悟该项目的各种设计思想与代码实现细节，基本定位是“精通”，精益求精，学无止境。这是大神们追求的境界。如果希望成为团队技术担当、项目社区的重要贡献者，则应当以这个层次作为努力的目标。

代码不仅是读，还要理和试，有的人连API都没有调用过，上来就看代码，以为省了时间，实际是迈向自我摧残。

对源码进行理和试的关键如下：

- 在IDE拿到调用栈：在IDE里读。IDE里可以方便跳转，查看定义，比起网页上看效率高得多。 通过IDE工具，运行example程序进行跟踪调试，通过打断点可以得到程序运行的调用栈。 尽可能编译调试。能调试的代码，几乎没有看不懂的。
- 把调用栈画下来：把代码的调用逻辑梳理出来之后，再通过画图工具，把代码的图画出来，可以画：流程图、类图、调用图、时序图，更具实际情况选择最有表现力的图。

此外，平时多了解一些设计模式。这样看到名字里有 proxy,builder,factory 之类的，就心领神会了。横向分层，纵向分块。代码都是分模块的，有的是 core,有的是 util，parser 之类的，要知道看的是哪一层，那一块。

有的小项目分层不明显，也不必强求。要看的不只是语法上的技巧，更重要的是设计上的思路和原理。读没读懂，最简单的标准是，假如给充足的时间，有没有信心写出一个差不多的东西来。
## 总结
学习开源，首先要明白为什么学习，不能为了学习而学习，这样是很容易放弃的，对于我而言，学习开源项目的动力就是：

- 面试需要
- 解决问题的能力提升
- 站在巨人的肩膀上，比自己摸索写代码成长快了很多
- 提升系统架构能力和掌握通用性的知识体系

实际实践操作中，完整执行上面 4 个步骤花费时间就长，通常情况下，前面 2 个步骤，在研究开源项目的时候都必不可少，第 3 个步骤可以在工作中打算采用开源项目才实施，第4个步骤在有一定的时间和精力下灵活安排时间做。

与其每个项目走马观花去简单了解，不如集中火力把一个项目研究吃透，即使半年才吃透一个，积累几年之后数量还是很可观的。而且很多项目的思想是共同的，例如高可用方案、分布式协议等，研究透一个，再研究类似项目，会发现学习速度非常快，因为已经把共性的部分掌握了，只需要再研究新项目差异的部分。

同时，在学习的过程中，需要不断总结，复盘，输出学习笔记，一方面锻炼逻辑思维能力，一方面有利于建立知识索引，过一段时间回顾的时候通过索引可以快速重新掌握知识，不容易遗忘。

## Refer
[谈谈如何高效学习开源项目](https://juejin.im/post/5b74031c518825612c20e8be)
[如何去阅读并学习一些优秀的开源框架的源码？](https://www.zhihu.com/question/26766601)