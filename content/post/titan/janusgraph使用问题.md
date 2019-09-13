---
date: 2018-04-26
title: "janusgraph一次给janusgraph提交源码的过程"
author: "邓子明"
tags:
    - 源码
    - janusgraph
categories:
    - 源码分析
comment: true
---

#
研究了好久的 neo4j源码，现在公司要换 janusgraph，只要半途而废开始研究 janusgraph 了
`https://github.com/JanusGraph/janusgraph`和`http://janusgraph.org/`



## 一、相关问题

https://github.com/JanusGraph/janusgraph/issues/1157

reindex 的时候，一直等待三分钟。并且打印日志：
""2018-06-10 09:03:19 [Thread-15] ERROR o.j.g.d.management.ManagementLogger - Evicted [6@6d56b8c524955-pc-jblur-com3] from cache but waiting too long for transactions to close. Stale transaction alert on: [standardjanusgraphtx[0x67a3ba21], standardjanusgraphtx[0x6cf78315], standardjanusgraphtx[0x48ce7bcd], standardjanusgraphtx[0x1862c45e], standardjanusgraphtx[0x04c1309d], standardjanusgraphtx[0x13bda0b2], standardjanusgraphtx[0x1187c9e8]]

实际上原因是
There is a bug when we are reindexing a new index in an empty database. The process tries to fetch data from the database but there are no data. Because of that, the process tries to get some data for 3 minutes with the similar error logged as this one:

我已经做了修复并提交到 janusgraph 的源码，等待merge。
https://github.com/JanusGraph/janusgraph/pull/1162

现在已经被合并了，我也成了janusgraph的contributor