---
date: 2018-04-22
title: "es架构-2"
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

## 一、Anatomy of an Elasticsearch Cluster -2

上一节我们说了：underlying storage model and CRUD operations in Elasticsearch.这一节的内容主要包括：
Consensus — split-brain problem and importance of quorum
Concurrency
Consistency: Ensuring consistent writes and reads
Translog (Write Ahead Log — WAL)
Lucene segments



### 1. Consensus- Split-brain problem and importance of quorum

Consensus 算法包括 Raft、Paxos等，为了解决一致性问题，es的一致性算法有两个部分：

Ping: The process nodes use to discover each other
Unicast: The module that contains a list of hostnames to control which nodes to ping

es是一个P2P的系统，所有节点都和其他节点沟通，有一个主节点，控制和更新集群操作。一个新的集群需要经过选举，一个节点被选为master，其他的加入master。
As nodes join, they send a join request to the master with a default join_timeout which is 20 times the ping_timeout. 
如果mster节点挂了，cluster重新开始ping，开始新的选举。这种ping过程也帮忙解决脑裂（某个节点突然觉得maste挂了，开始寻找新master）

为了容错，master会ping 所有的 节点去检查是否 alive然后节点会ping master进行response。
默认配置下，es可能会有脑裂， 由于 network partition,a node 觉得 master 已经 failed然后自己当上master，导致连个master。
This may result in data loss and it may not be possible to merge the data correctly. 
This can be avoided by setting the following property to a quorum of master eligible nodes.
`discovery.zen.minimum_master_nodes = int(# of master eligible nodes/2)+1`

设置这个配置之后，需要有 quorum of active master eligible nodes 参加完成master选举过程，并接受他的master身份。
This is an extremely important property to ensure cluster stability and can be dynamically updated if the cluster size changes. 
NOTE: For a production cluster, it is recommended to have 3 dedicated master nodes, which do not serve any client requests, out of which 1 is active at any given time.

这就是 Consensus 的内容

### 2. Concurrency

对于高并发，es使用 optimistic concurrency control 乐冠锁进行控制，保证新纪录不被就记录覆盖。
每个document indexed 有一个 version number which is incremented with every change applied to that document。保证每次更新都能按照顺序。
为了保证数据不丢失，es可以让你自己指定id，如果你指定的id比present的小，更新就失败了。
How failed requests are handled can be controlled at the application level. 
There are also other locking options available and you can read about them：https://www.elastic.co/guide/en/elasticsearch/guide/2.x/concurrency-solutions.html.

### 3. Consistency — Ensuring consistent writes and reads

写一致
对于怎样算写成功，可以设置 available的 shards 数量
The available options are quorum, one and all. By default it is set to quorum and that means that a write operation will be permitted only if a majority of the shards are available.

尽管大多数available，也有可能出错。the replica is said to be faulty and the shard would be rebuilt on a different node.

读一致：
new documents are not available for search until after the refresh interval. 
为了保证读到最新的document，replication can be set to sync (default) 。这样只有 primary and replica shards 都写完了才会返回 write request 。
这样，从任何一个shard查询，都将返回最新的document。
Even if your application requires replication=async for higher indexing rate, there is a _preference parameter which can be set to primary for search requests. 
这样，查询都走 primary shard ，保证结果都来自最新版本。



### 2. Translog


WAL来自关系型数据库的世界，translog保证事件失败时候的数据完整性，通过底层的原则 :在将数据提交到磁盘的实际更改之前，必须记录并提交预期的更改。

当新 document被index，或者旧的更改，Lucene index会改变，然后改变会被提交到磁盘进行持久化。每次update都进行提交很expensive，更好地办法是一次性提交很多。
上一节提到的，flush操作默认30min一次或者translog太大，这样的话，有可能丢失30min的数据。为了避免这个问题，es使用translog，update操作都将被写到translog，
translog is fsync’ed after every index/delete/update operation (or every 5 sec by default) 保证改变被持久化。
The client receives acknowledgement for writes after the translog is fsync’ed on both primary and replica shards.

当两次flush之间出现问题，translog将会重新执行，从而恢复丢失的change，
NOTE: It is recommended to explicitly flush the translog before restarting Elasticsearch instances, 
as the startup will be faster because the translog to be replayed will be empty. 
POST /_all/_flush command can be used to flush all indices in the cluster.

上面已经说了translog的flush操作，segments文件会被提交到磁盘来保证改变持久化。接下来我们看看什么是lucene的 segment。

### 3.Lucene Segments
Lucene索引由多个段组成，并且段本身是一个全功能倒置索引。
是不可变的，它允许Lucene在不从头开始重建索引的情况下增量地向索引添加新文档。
对于每一个搜索请求，索引中的所有段都被搜索，并且每个段消耗CPU周期、文件句柄和内存。这意味着段数越高，搜索性能就越低。
为了解决这个问题，es将小的段合并成更大的段，将新合并的段提交到磁盘并删除旧的较小的段。
对于搜索请求，搜索给定的弹性搜索索引碎片中的所有Lucene段，但是，在排名结果中提取所有匹配的文档或文档对于弹性搜索集群是危险的。


## What next?

在后续文章中，我们将看到这一点，并回顾下面的主题，其中包括在弹性搜索中进行的一些权衡，以在低延迟下服务相关搜索结果。

lit brain problem in Elasticsearch and how to avoid it
Transaction log
Lucene segments
Why deep pagination during search is dangerous?
Difficulties and trade-offs in calculating search relevance
Concurrency control
Why is Elasticsearch near real-time?
How to ensure consistent writes and reads?