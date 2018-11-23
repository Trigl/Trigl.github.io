---
layout:     post
title:      "Apache Calcite 学习"
date:       2018-08-13
author:     "Ink Bai"
catalog:    true
header-img: "/img/post/winter-akka.jpg"
tags:
    - Big Data
---

## 背景
Apache Calcite 是一个动态数据管理框架，它包含很多传统数据库具有的组件，但是省略了一些核心功能，如：数据存储、处理数据的算法和存储元数据的仓库。

Calcite 故意剥离出存储和处理数据的业务逻辑，仅仅作为应用和其他数据存储系统的媒介，可以更加容易的创建一个自己的数据库，你所需要做的就是添加数据。

为了解释地更加直观，让我们创建一个 Calcite 的实例然后往里面加一些数据。
