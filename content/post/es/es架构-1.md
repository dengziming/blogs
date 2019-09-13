---
date: 2018-04-22
title: "es架构-1"
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

## 一、Anatomy of an Elasticsearch Cluster

很遗憾，Google的搜索技术不开源，es是搜索引擎的一个很好的替代品，本文主要覆盖了es的底层结构、数据原型、读写过程。es的功能主要有：

1. 全文搜索
例如：怎么找到Wikipedia上面和某个名字最相关的文章

2. 聚合

例如 显示广告网络上面的词条出价直方图

3. 地理空间API

例如：设计一个能找到和骑手最近司机的骑行分享平台

接下来就是内容，主要有以下几个方面：
1. 是主从架构还是无主架构
2. 存储模型
3. 读写工作流程
4. 搜索结果怎么相关

### 1.The confusion between Elasticsearch Index and Lucene Index + other common terms…

es 的 index 是一个组织数据的逻辑空间，就类似一个数据库。es index有一到多个 shards，一个shard就是一个真正存数据的lucene index，内部就是一个搜索引擎。

每个 shard 都有0到多个replica，es index 有 type 的概念，就好比数据库里面的表，一个type里的所以type有相同的properties，就像schema一样。

### 2. Types of nodes

#### （1）Master Node

控制集群，负责集群的操作，例如创建删除index，和集群的nodes联系，给节点分配shards。主节点一次处理一个集群状态，并将状态广播到所有节点，收到广播的节点对主节点进行确认回复。
an be configured to be eligible to become a master node by setting the node.master property to be true (default) in elasticsearch.yml.
大集群最好有专门的master node，去空值集群，不用处理任何用户请求

#### （2） Data Node

保存数据和倒排索引，By default, every node is configured to be a data node and the property node.data is set to true in elasticsearch.yml. 
If you would like to have a dedicated master node, then change the node.data property to false.

#### （3）Client Node:

If you set both node.master and node.data to false, then the node gets configured as a client node and acts as a load balancer routing incoming requests to different nodes in the cluster.

#### （4）coordinating node

注意没有专门的coordinating node，通过client连上的的es节点称为 coordinating node，将client request 路由到合适的shard。对于读请求，每次选择不同的shard 从而 balance the load.

### 2. Storage Model

Elasticsearch uses Apache Lucene, a full-text search library written in Java and developed by Doug Cutting (creator of Apache Hadoop)。
es内部通过倒排索引的数据结构，从而处理可能延迟的查询。es中 document 是数据的存储 unit，通过将document的词进行分词创建 inverted index ，倒排索引能够创建排序的term并将和这个term相关的document进行管理。
和每本书背后的index类似，包含了很多词和那一页可以找到这些词，例如下面的两个document。

Doc 1: Insight Data Engineering Fellows Program 
Doc 2: Insight Data Science Fellows Program

If we want to find documents which contain the term “insight”, we can scan the inverted index (where words are sorted), find the word “insight” and return the document IDs which contain this word, which in this case would be Doc 1 and Doc 2.

为了更好的搜索性，文档先被分析。一般就是分词+标准化。

综上，我们知道每个document存储模型，存储了document，以及对他们分词后的倒排索引。

### 3.Anatomy of a Write
1. (C)reate

When you send a request to the coordinating node to index a new document, the following set of operations take place:

es所有的节点都包含了集群的元数据信息，包括哪些节点或者，有哪些分片。The coordinating node 通过 documentId将document和他对应的shard route起来，
es再通过 murmur3 hash算法将documentId进行取值，得到shard。`shard = hash(document_id) % (num_of_primary_shards)`。
当节点收到 coordinating node 的 request ，request 会被写入到 translog 中，document 会被放进 memory buffer（http://www.linfo.org/buffer.html），
如果在primary shard上执行成功，reques也会被发送到 replica shard上，
当 translog fsynced （https://linux.die.net/man/2/fsync） on all primary and replica shards.client receives acknowledgement that the request was successful。

memory buffer会周期性更新 (defaults to 1 sec)，contents 会被写到一个 a new segment in filesystem cache ，
This segment is not yet fsync’ed, however, the segment is open and the contents are available for search.

The translog is emptied and filesystem cache is fsync’ed every 30 minutes or when the translog gets too big. 这个过程称为flush
the in-memory buffer is cleared and the contents are written to a new segment.
A new commit point is created with the segments fsync’ed and flushed to disk. The old translog is deleted and a fresh one begins.

2. (U)pdate and (D)elete

es的记录是无法更改的，删除和update实际上是新建，更改版本号。每个segment 都有一个 .del  file。
When a delete request is sent, the document is not really deleted, but marked as deleted in the .del file. 
This document may still match a search query but is filtered out of the results.
When segments are merged, the documents marked as deleted in the .del file are not included in the new merged segment.

update则是新建+删除，es给每个document一个version，每次改变，version都+增加，旧版本会被.del 文件标记为删除，
和删除一样，This older document may still match a search query but is filtered out of the results.

3. Anatomy of a (R)ead
Read operations consist of two parts:

Query Phase
Fetch Phase

- Query Phase
coordinating node route the search request to all the shards (primary or replica) in the index. 
每个shard单独search，然后将结果放进一个优先队列，根据 relevance score (we’ll cover relevance score later in the post).
所有 shards将结果汇总，创建一个新的优先队列，取出相关度最高的一部分。这个过程类似spark的topn

- Fetch Phase
 coordinating node 排好序之后，
it then requests the original documents from all the shards. All the shards enrich the documents and return them to the coordinating node.

相关度：tf/idf （term frequency/inverse document frequency）算法。
tf 词频，在某文档出现的频率
idf 出现过得所有文档数

## What next?

lit brain problem in Elasticsearch and how to avoid it
Transaction log
Lucene segments
Why deep pagination during search is dangerous?
Difficulties and trade-offs in calculating search relevance
Concurrency control
Why is Elasticsearch near real-time?
How to ensure consistent writes and reads?