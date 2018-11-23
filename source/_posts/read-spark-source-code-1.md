---
layout:     post
title:      "Spark 源码阅读（I）"
subtitle:   "通过 Spark Submit 提交应用流程分析"
date:       2018-06-06
author:     "Ink Bai"
catalog:    true
header-img: "/img/post/spark-code-1.jpg"
tags:
    - Spark
---

> 提交 Spark 应用的第一步是通过 spark-submit 执行的，本文将会从源码研究整个过程，对源码的详细注释可以查看这里： https://github.com/Trigl/spark

我们用 Spark 自带的 `SparkPi` 的例子来讲解，我们通过 spark-submit 提交的命令如下：

```
./bin/spark-submit --class org.apache.spark.examples.SparkPi \
--master yarn \
--deploy-mode cluster \
--driver-memory 2g \
--executor-memory 1g \
--executor-cores 1 \
--conf spark.yarn.jar=hdfs://ip-10-201-1-216.cn-north-1.compute.internal:8020/apps/spark/spark-libs.zip \
examples/jars/spark-examples.jar \
10
```

bin 目录下的 spark-submit 脚本首先检测一下 `SPARK_HOME` 是否设置，然后会把所有的参数传给 bin/spark-class 去执行，spark-class 才是实际执行的脚本，关键代码如下：

```bash
# `$@` 是指传入该脚本的所有参数
# 在这里把参数传递给了 `spark-class` 实际执行，并且传入了 `org.apache.spark.deploy.SparkSubmit` 这个类
exec "${SPARK_HOME}"/bin/spark-class org.apache.spark.deploy.SparkSubmit "$@"
```

spark-class 做了什么呢？第一步还是检查 `SPARK_HOME`，然后执行 `load-spark-env.sh`，这个脚本的作用是加载 `spark-env.sh` 的环境变量，然后找 `JAVA_HOME`，然后查找 Spark 的 jar 包作为 classpath，到此为止程序执行的准备工作已经完毕了，接下来做的事情是检查并构建规范化的参数，因为我们从终端输入的命令中，可能有的参数是错的，这就需要首先检查一下这些参数是否正确，如果正确还需要规范化成我们需要的样式。
