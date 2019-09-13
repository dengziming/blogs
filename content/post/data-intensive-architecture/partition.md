---
date: 2018-12-22
title: "分布式存储-partition"
author: "邓子明"
tags:
    - 架构
    - 大数据
categories:
    - 设计高并发架构
comment: true
---


# 二、Partitioning

分区，also known as Sharding、region、tablet、vBucket

partition 的主要目的是 scalability，其实相对于 其他分布式理论，partition 是最简单的，没有很多复杂的一致性问题。