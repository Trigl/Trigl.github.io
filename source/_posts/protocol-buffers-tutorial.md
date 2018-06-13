---
layout:     post
title:      "Protocol Buffers 了解一下？"
date:       2018-06-13
author:     "Ink Bai"
catalog:    true
header-img: "http://ox2ru2icv.bkt.clouddn.com/image/post/protocol-buffers.jpg"
tags:
    - Big Data
---

> 本文是学习 Protocol Buffers 时做的笔记，内容来自[官方文档](https://developers.google.com/protocol-buffers/)，把其中的精华部分提炼出来做了一个总结。

## 什么是 protocol buffers
Protocol buffers 是一种跨语言跨平台可扩展的序列化结构化数据的方式，常用于通信协议、数据存储等等。首先会定义数据应当如何构造，然后使用特殊生成的源代码把结构化的数据写入到各个数据流或者读取出来。
