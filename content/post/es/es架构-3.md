---
date: 2018-04-22
title: "es架构-3"
author: "邓子明"
tags:
    - 架构
    - 大数据
categories:
    - es
comment: true
---

es设计架构，良心参考资料：
https://blog.insightdatascience.com/anatomy-of-an-elasticsearch-cluster-part-i-7ac9a13b05db
https://blog.insightdatascience.com/anatomy-of-an-elasticsearch-cluster-part-ii-6db4e821b571
https://blog.insightdatascience.com/anatomy-of-an-elasticsearch-cluster-part-iii-8bb6ac84488d

## 一、Anatomy of an Elasticsearch Cluster -3

上一节我们说了：how Elasticsearch approaches some of the fundamental challenges of a distributed system.这一节的内容主要包括：

Near real-time search

Why deep pagination in distributed search can be dangerous?

Trade-offs in calculating search relevance



### 1. Near real-time search

尽管改变不会立即可见，但是es确实提供了近实时查询，前面的内容提到，lucene的segment改变提交到磁盘是很消耗性能的。

为了避免search的同时提交改变到磁盘，es在memory buffer和disk之间提供了一个 filesystem cache，memory buffer 默认1s refreshed 一次，
一个包含倒排索引的segment也会在 filesystem cache中生成。segment是开放的可以查询。

filesystem cache有文件句柄而且可以打开，读写，关闭，尽管它在内存中。
Since, the refresh interval is 1 sec by default,  the changes are not visible right away ，所以是准实时。
Since, the translog is a persistent record of changes not persisted to the disk, it also helps with the near real-time aspect for CRUD operations.
在查找相关段之前，在 translog 中搜索任何最近的变化，因此，客户端可以访问近实时的所有变化。

你可以每次更新后手动刷新index，但是这样会产生很多小segment，不推荐。
For a search request, all Lucene segments in a given shard of an Elasticsearch index are searched。
however, fetching all matching documents or documents deep in the resulting pages is dangerous for your Elasticsearch cluster. Let’s see why that is.


### 2. Why deep pagination in distributed search can be dangerous?

当我们查到的es中有很多匹配到的document，默认返回最相关的10条，分页操作会提供from和size两个参数，然后每个shard上面会产生from+size个结果放进优先队列，最后汇总。

加入你需要的是10000到100010个结果，每个shard的优先队列会返回10010个结果排序后放进 memory中，这样将会有很大的隐患。
scroll API （https://www.elastic.co/guide/en/elasticsearch/reference/current/search-request-scroll.html）可以让你返回所有的结果。

前面说到es使用tf-idf算法，分布式系统计算idf时很麻烦的，需要有aggregate操作。es的做法是返回一个local idf：
Instead, every shard calculates a local idf to assign a relevance score to the resulting documents and returns the result for only the documents on that shard. 
Similarly, all the shards return the resulting documents with relevant scores calculated using local idf and the coordinating node sorts all the results to return the top ones。
大多数情况可靠，但是数据倾斜时候就不太可靠了。

对于上面的问题有两种 trade-off，都不太适合大规模数据。一种是只有一个shard，这样local idf就是 globe idf。另一种是 dfs_query_then_search，先把local idf合并成globe idf，然后在计算。

## What next?

In the last few posts, we reviewed some of the fundamental principles of Elasticsearch which are important to understand in order to get started. 

接下来，我们要看看他的源码，或者看看怎么和spark、hadoop结合使用。