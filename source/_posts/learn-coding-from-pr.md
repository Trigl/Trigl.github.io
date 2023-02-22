---
layout:     post
title:      "从一次开源项目的 PR 学习如何写代码"
#subtitle:   "实现 Full Checkpoint"
date:       2023-02-22 10:00:00
author:     "Ink Bai"
catalog:    true
header-style: "text"
#header-img: "/img/post/lakehouse.jpg"
tags:
    - 因崔斯汀
---

最近由于工作原因我需要将 PrestoDB 社区一个比较新的功能特性 backport 到我司内部的代码中，然后很快找到了这个 [PR](https://github.com/prestodb/presto/pull/16999)，但是打开之后，我惊讶的发现这个 PR 尽然有 6000 多行代码和 80 个讨论。

![](/img/content/screenshot-20230222-232116.png)

于是带着探究的心态去看了一下具体的讨论内容，发现讨论的内容并没有涉及很多设计方面的东西，更多的是很微小的代码细节，可以说这是一次非常经典的 code review 的案例：

- 开发者工作经验不是太多，犯了很多新手会犯的问题
- reviewer 经验很丰富，review 的非常细致，并且给出了非常具体的指导

一起来看看我们能从这个 PR 中学到什么。

#### 尽量写 `不可变` 的代码
![](/img/content/screenshot-20230222-233152.jpg)

注意这里这个类开头加上了 `@Immutable` 的标签，因此要保证类是不可变的。也就要保证这个类初始化后，所有的成员变量都不能再发生改变。开发者考虑到了不添加 `setter` 方法，但是忘记了 List 类型的成员变量传递的是引用，还是可以发生变化，而是应当使用 `ImmutableList` 保证不变。

![](/img/content/screenshot-20230222-235548.png)

同样，如果方法返回的结果可能会发生变化，并且变化了之后可能会产生副作用，那就直接返回一个不可变的结果。

![](/img/content/screenshot-20230223-000941.png)

成员变量尽量定义为 `final` 的

#### 学好 Java 第一步，避免空指针
![](/img/content/screenshot-20230222-234440.png)

#### 如何使用 `static` 关键字
什么时候应该使用 `static`：
![](/img/content/screenshot-20230223-002307.png)

什么时候不应该使用 `static`：
![](/img/content/screenshot-20230222-234930.png)

#### List 要带上具体的泛型
![](/img/content/screenshot-20230222-235158.png)

哈哈，这个是非常典型的新手写法。

#### 要经常考虑代码的性能
空间换时间：
![](/img/content/screenshot-20230223-000343.png)

通过加限制条件避免频繁执行某一逻辑：
![](source/img/content/screenshot-20230223-002936.png)

#### `log.error` 也可以打堆栈！
![](/img/content/screenshot-20230223-001513.png)

成年人要学会放弃 `System.out.println()` 和 `e.printStackTrace()` 了！

#### 不要重复造轮子
![](/img/content/screenshot-20230223-001809.png)

对于一个成熟的开源项目来说，一定会存在大量基础的工具类供我们调用，所以增加新的代码模块可以先看看之前是不是有类似的代码可以借鉴，而不是直接用新的轮子，这样第一可以提升编码效率、第二可以统一项目的技术栈和代码风格。

#### 通过 shaded jar 的方式解决冲突
![](/img/content/screenshot-20230223-003531.png)

为了避免 pom 依赖中 jar 太多难以管理，并且解决 jar 冲突问题，一般会另外引入一个只包含 pom 文件的 shaded 的项目。

怎么样，看完了这个 PR 的 code review 之后有何感受？是不是有种给实习生看代码的感觉？但是你能保证自己能一直像这个 reviewer 一样负责任有耐心吗，这也许才是我们应该思考和学习的。

如果现实条件不允许每次都这么详细的做 code review，那么没事看看开源 PR 的讨论和代码实现，也不失为一种好的学习方式。
