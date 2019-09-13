---
date: 2018-12-22
title: "design-data-intensive-architecture"
author: "邓子明"
tags:
    - 架构
    - 大数据
categories:
    - 设计高并发架构
comment: true
---


# 第一部分、前言

ACID，bigdata，NoSql ，bigTable ，CAP，2PC，3PC，quorum，raft，paxos，cloud，real-time，高并发架构将会逐渐成为基础。

主要内容：
（一）基础

1. “reliability, scalability, and maintainability”
2. 数据模型
3. 存储引擎
4. 数据结构

（二）分布式

1. replication
2. partitioning/sharding
3. transaction
4. distributed system
5. consistency and consensus

（三）交互

1. batch process
2. realtime
3. put everything together

（四）案例及资料



# 第二部分、数据系统的基础


## 一、Reliable,Scalable 和 Maintainable 的含义

数据系统分为 data-intensive 和 compute-intensive，一般的大数据应用会采用的技术：
1. 存储，方便后续的查询 -- databases
2. 缓存，加速访问 -- caches
3. 允许用户通过关键字搜索、按照过滤  -- indexes
4. 给另一个进程发送消息，异步处理  -- streaming process
5. 周期性处理全量数据  -- batch process

这一切都是需要数据系统，我们并不会从头开始搭建存储、计算系统，因为有很多现成的技术可以应用。数据库、消息队列功能都很相似，但是实现完全不同，导致不同的性能，所以我们用处不同，我们为什么要将他们结合到一起呢？

其实许多工具功能比较重复，redis 其实也可以当消息队列，kafka 也可以做数据持久化。这些工具的界限越来越模糊。第二是我们得系统一般比较复杂，一个工具完成不了，我们必须组件一个数据系统。组件的过程
有很多需要考虑的事情，其中最重要的三点：
Reliable： 系统必须一致能够提供服务，即便是发生了一些不可预知的错误
Scalable： 系统升级、功能扩展、访问量增加，导致我们需要能够很方便扩展我们的系统
Maintainable： 随着时间推移，可能有很多其他人接受项目，必须方便别人进行维护

有关这三点的介绍，我们逐渐展开：

### 1. Reliable

### 2. Scalable

### 3. Maintainable

## 二、数据模型和查询语言

数据模型是最重要的一部分，

### 1.关系型与文档型

### 2.NoSQL

### 3. Object-Relational Mismatch

### 4. Many-to-One and Many-to-Many Relationships

### 5. Query Languages for Data

### 6.Graph-Like Data Models


## 三、Storage and Retrieval

### 1. Hash Indexes

### 2. SSTables and LSM-Trees

### 3. B-Trees

### 4. Multi-column indexes

### 5. Full-text search and fuzzy indexes

### 6.Keeping everything in memory

### 7. OLAP vs OLTP

### 8. 数据仓库

### 9. Stars and Snowflakes: Schemas for Analytics


### 10. Column-Oriented Storage

### 11. Column Compression

### 12. Sort Order in Column Storage

### 13. Writing to Column-Oriented Storage

前面写的有关列存储的优化在数据仓库中很重要，排序、位图索引、压缩都能加速查询，但是会让写数据更加麻烦。

以 BTree 为理论的写方法，在这里完全没有用，如果你要在数据中插入一条数据，你需要重写所有的数据。

但是还好的是，如果是 LSM-tree 的方式，首先写进内存排序，然后写磁盘。查询的时候，也是两部分结果进行合并，Vertica 数据仓库就是这么做的。

### 14. Aggregation: Data Cubes and Materialized Views

并不是所有的数据仓库都是使用列存储，很多其他存储方式也会用到，但是列存储确实能够很大程度加快访问，所以越来越流行。

另一种值得一提的数据仓库技术就是 materialized aggregates。前面所说的，查询经常会遇到一些聚合， 例如 count，sum，如果一个查询被很多查询共用，可以将结果缓存。
其中一种方法就是 materialized view，类似查询视图，只不过这个视图的结果已经计算好保存起来了。如果数据改变了，你的 materialized view 也要改变。

一种常见的案例就是 data cube 后者 OLAP cube，就是根据不同纬度就行聚合后的结果。举个列子，如果表格有 日期、商品种类、销售额三列，我们可以计算销售额在 日期、商品种类 两个维度下的聚合只，得到一个二维表格。
二维表格的两个维度分别为日期和商品种类，表格的值就是销售额，然后如果要计算每个维度下的 sum，只需要将对应维度所有值 sum 到一起。

实际上，表格都不止两个维度，假如有五个维度，情况就很复杂了。但是原则不变，每个 cell 保存五个维度组合下的聚合值。

通过 cube 可以加速某些查询，但是如果要计算订单额度大于 100 的比例，那就没法计算了，因为额度很难作为一个维度。所以一般只作为一个加快部分查询的工具。

## 四、Encoding and Evolution

服务总是要变的，服务变了，服务之间的代码也要变。服务端可以做滚动升级，也就是部分服务器先升级，保证没错的话，其他的继续升级。
而客户端也就是用户端，代码变化必须做到向前向后兼容，也就是说，高版本要能够处理低版本的数据，这很简单；低版本的代码，必须能从高版本的架构读数据，这是比较难的。
接下来我们将会看一些数据结构，JSON, XML, Protocol Buffers, Thrift, and Avro ，主要看他们在这些data storage and for communication系统中如何使用：
in web services, Representational State Transfer (REST), and remote procedure calls (RPC), as well as message-passing systems such as actors and message queues 

### 1. Formats for Encoding Data

一般的应用都会处理两种数据。内存主要是通过对象、数组、树、hash表等，而需要保存、传输时候，需要进行序列化，由于指针引用序列化后是没用的，所以序列化后的结构和内存结构完全不用。
从设计模式角度考虑，我们需要进行一个适配，两边都要适配的话，实际上就是做一个转换。内存到 byte sequence 的过程称为 encoding（或者 serialization、marshalling），反过来讲decoding（deserialization、parsing、unmarshalling）

** 专业术语冲突
serialization 在数据库的事务中也会用到，但是完全是另外一码子事情，后续会进行介绍。这里使用的时候我们以 encoding 为主，虽然平时 serialization 用的更多。

这里我们将会以一条数据为例,对它进行编码：

```json
{
    "userName": "Martin",
    "favoriteNumber": 1337,
    "interests": ["daydreaming", "hacking"]
}
```

#### (1) Language-Specific Formats

java.io.Serializable 属于java 自带的序列化机制，很多编程语言都自带了。当然也有很多第三方的，比如 Kryo for Java。这些很方便，因为你只需要直接读然后调用java对应的方法即可。但是问题也很多。

1. 通用性问题，不多说。
2. 数据会有类型，例如java 序列化必须只能解析为特定的类。所以会定义很多类，这样可能导致一些人得到你的类信息，然后远程植入代码。
3. 版本问题。
4. 效率问题。

#### (2) JSON, XML, and Binary Variants

JSON, XML 一般都很熟悉，优缺点都很明显，二进制的编码在某些内网的应用中很常见，还能节约空间。例如 MassagePack 就是二进制的 json。

#### (3) Thrift and Protocol Buffers

Thrift and Protocol Buffers 很多人都知道，作为两种序列化的框架，当然还提供了别的功能，例如 RPC 通信接口。都需要定义一个schema：

Thrift 的定义格式：

```java
struct Person {
  1: required string       userName,
  2: optional i64          favoriteNumber,
  3: optional list<string> interests
}
```

pb 的定义格式：

```java
message Person {
    required string user_name       = 1;
    optional int64  favorite_number = 2;
    repeated string interests       = 3;
}
```

然后他们都有代码生成的工具，可以针对各种语言生成相应的代码，用来 encode 和 decode 数据。thrift 的格式有两种 BinaryProtocol and CompactProtocol。

BinaryProtocol 表示上面的数据格式需要 59 bytes ：

```
0b 00 01 00 00 00 06 4d 61 72 74 69 6e     0b 是 type代表 String，紧接着 00 01 代表 field tag 为1，然后 00 00 00 06 代表长度为6，剩下的六bytes 代表数据。
0a 00 02 00 00 00 00 00 00 05 39           和刚才一样， 0a 代表int64 ，
0f 00 03 0b 00 00 00 02                    0f 代表list，00 03代表 field tag 为3，0b 代表 item type 为 string，00 00 00 02 代表长度为2，
	00 00 00 0b 64 61 79 ....              长度和数据
	00 00 00 07 68 ..... 				   长度和数据
00											结束标志
```

CompactProtocol 和 BinaryProtocol 的结构类似，但是进行了一些压缩，需要 33 bytes

```
18 06 4d ....  							  18 是 00011000 field type and tag number 合在一起，0001 是 tag，
										  1000 是 type 是string （为什么8 代表 string，上面是11），06 长度，后面是数据
										  另外int类型并不是占了 64位，这里的 06是两位，共 8 字节，
										  最高位代表数据是否已经完整，剩下的七位是数据。如果数据不完整，需要再占8位。这样 -64 到 63 之间的数据只占了1byte
										  
16 f2 14 								  16 是 00010110，0001 代表 field tag 比上一个加一，0110 数据类型 后面的 f214 是数据，f2 最高位是1，数据没完，需要往下一位读。
19 28
    0b 64 .....
    07 68 ....
00
```

ProtocolBuffer 和 Thrift 的 CompactProtocol 类似，需要 33 bytes

```
0a 06 4d 61 72 74 69 6e       			  0a 是 tag(00001) 和 type(010), 06 是长度。 后面是数据
10 b9 0a 								  10 是tag (00010) 和 type（000）,后面是长度+数据
1a 0b ........							  1a 是 tag 和 type，后面是长度+数据
1a 07 ........							  1a 是 tag 和 type，后面是长度+数据
```

和 CompactProtocol 稍微有些不一样，tag+type 为两位，另外 list 类型直接存两个。

需要注意的是，前面我们的例子中，数据要么是 要么是 required or optional，实际上这对于数据的存储没有任何影响，都是一样的存储，只是在最后解析的时候影响，例如你设置的是 required ，但是没有解析出来，会失败。

#### (4) Field tags and schema evolution

前面讲过数据需要改变，那 pb 和 thrift 如何应对schema 的变化，做到向前向后兼容呢？
其实我们从上面的数据可以看出，最重要就是 field tag 了，只要 field tag 唯一，哪怕是增加了数据，减少了数据，最终的结果其实还是一样的。你可以改变变量名字，但是不能改变 tag。
首先是向后兼容，只要你新加的 field tag 不被设置为 required，就不会报错，另外删除数据也是，不能删除 required 的数据。
另外是向前兼容，如果是旧的代码读取新的数据，如果遇到不认识的 field tag ，会直接忽略。

#### (5) Datatypes and schema evolution

数据类型发生变化，一方面可能是数据精度收到影响，另一方面，我们可以看出 pb 是可以讲啊 list 类型变成 optional 的，但是 thrift 不行，因为 thrift 提供了 list 类型，但是这样可以用来写嵌套数据。

#### (6) avro

avro 也是一个序列胡框架，他和 pb、thrift 不同点在于没有 tagNumber，这其实很适合动态生成schema，我们不介绍了。

#### (7) Dynamically generated schemas

如果我们的数据增加了一列，并且减少了一列，对于 avro 来说，只需要重新生成一个 schema 即可，而如果使用 pb、thrift，需要手动进行配置tag和field 的映射，

#### (8) Code generation and dynamically typed languages

代码生成，例如 pb 可以生成 java 和 python 的代码。

#### (9) The Merits of Schemas

### 二、 Modes of Dataflow

我们需要在进程之间发送数据，

#### 1.Dataflow Through Databases



#### 2.Dataflow Through Services: REST and RPC

#### 3.  Message-Passing Dataflow

# 第二部分、分布式

使用分布式的原因很多，例如 Scalability，high available，latency。数据分布式有很多种方式，首先是副本、分区，也就是 Replication 和 Partition。


# 一、Replication

Replication 就是将你的数据放在多个节点，这样有很多好处。数据从地理上可以隔用户更近，保持高可用，增加吞吐量。

本章我们假设数据都是完整的进行副本存放，后面再讲分区。数据的副本解决难点就是数据变化的一致性，如果数据一致不变，那就太简单了，没什么可以讨论的。

对于数据变化的副本存储，有很多需要注意的地方，例如每次写数据操作数据复制是同步的还是异步的，节点挂了怎么办，数据一致性问题，时效性问题。
我们的学习主要分为三种设计：single-leader, multi-leader, and leaderless

##  1. Leaders and Followers

每个节点上面的副本叫做 replica ，当有 multi replica 的时候，一个问题出现了，怎么保证数据写到了每一个副本。
数据库的每一次写都必须保证被每个副本处理，否则就会出现不一致。
最常用的解决方案就是 leader-based-replication，也叫 active-passive、master-slave。

在主从结构中，写只能发送给 leader，leader 写的时候会发送数据给 follower，而读操作可以读任何一个节点。

### (1) Synchronous Versus Asynchronous Replication

replicated 系统重要特点就是 同步写还是异步写，一般关系型数据库可以配置，其他的都是硬编码写死的。

客户端发送数据给leader，然后leader转发给follower，最后通知client。如果 leader 收到 follower 的确认消息再回复client，这就是同步。
如果leader发送给了 follower后就直接回复client，就是异步。一般情况，系统都只有一个follower是同步，其余的都是异步。这叫 semi-synchronous。

为了效率经常设置为 Asynchronous，这样如何 leader 失败了而且不恢复，那还没有被副本保存的数据就丢失了，
这样的好处就是可以提供持续服务，尽管follower很慢。即便是配置使用 Synchronous，实际上也是有一台是同步，其余的是异步。

Weakening durability 可能是一个比较糟糕的 trade-off，但是 Asynchronous 还是很常用，尤其是地理上分布式的集群。

当然也有很多其他的方案在研究中，例如 Azure 使用了 chain replication。

### (2) Setting Up New Followers

系统有时候需要添加新的节点，如何保证新节点和leader的数据一致呢？直接复制数据过来肯定是不行的，leader 一直处在 flux 中，一般步骤有四步：
1. leader 在某个确定一致性的时间点拍一个 snapshot 而不用锁住系统，大部分数据库都提供了这个功能，有的能通过第三方工具提供这个功能。
2. follower 将数据复制过去。
3. follower 将从这个时间点以后的数据复制过去，这需要拍 snapshot 时候的日志位置，这个点的名字叫 Log sequence number 或者 binlog coordinates。
4. 当 follower 将所有的change都同步后，我们称之为 catch up，就可以和其他follower一样，处理 leader 发生的变化了。

真正的步骤其实是有很多不同，有时候可以自动配置，有时候需要手动配置。

### (3) Handling Node Outages

有时候节点会挂掉，可能是我们不知道的原因，也可能是为了安装系统模块而重启，这时候应该怎么保证集群的高可用

1. follower 挂掉

follower 挂点然后重启的步骤是很简单的，类似前面的添加新节点，启动以后，根据日志能找到最后一个处理的事务，然后从leader请求这个时间点以后的更改，然后进行更改，完成后就 catch up。

2. leader failover

leader 挂掉后可能麻烦一点，必须马上选择一名新的leader，这个过程被称为 failover

failover的具体步骤有4：
1. 确定leader失败了，失败的原因很多，磁盘、电源等等，目前用的最多的是 timeout 方法，节点相互 bounce message，如果突然在某个时间点收不到，就是失败了。
2. 选择新的 leader，需要在大多数的follwer 进行选举，最好是选一个数据最新的。所有的节点同意这个节点成为 consensus。
3. 告知系统使用新的leader，client 发送数据到新的 leader，旧的 leader 回复后也要认同新的 leader。

failover 总是会有些不期而遇的问题：
1. 如果是 asynchronous replica ，可能遇到一些数据没有同步过来，导致数据少了，如果这时候 older leader起来了，就会发生错误，这时候一般选择 discard 这部分数据。
2. 如果系统和别的系统进行结合，discard 是很危险的，github 发生了一次 mysql leader挂掉，然后discard 了一部分，删掉了部分自增的primary key，可是这部分pk已经保存到 redis了，导致了错误
3. 有时候两个节点都认为自己是 leader，这个叫做 split brain，这时候一半需要去掉一个，但是有可能两个都被去掉了。
4. timeout 的具体值多少合适？如果太长了，可能导致发现的晚。如果太短了可能误判，尤其是系统压力大的时候，还重新换一次会造成更大的压力。

这些问题解决起来还是很棘手的，所以有很多算法，我们后续会介绍。

### (4) Implementation of Replication Logs

上面各种恢复，添加，都涉及到日志，日志记录了数据的更改，实际上我们学习 Hadoop 的时候，也知道里面有日志，现在我们来研究一下，日志怎么实现。

1. Statement-based replication

基于 statement 的 replication，这是最简单的方法，数据库记录每一次更新，也就是说，你的 UPDATE、INSERT、DELETE都会记录到日志中，然后发送给follower。
这样的问题是：
对于 current_timestamp、random 这样的方法返回值不一样，基于判断的例如 update ... where ... 必须按照顺序，一些触发器、存储过程等 side effects。

对于这些问题，当然也有特定的解决方案，例如先计算一些可能导致不一致的函数。

2. WAL-log

前面讲过 B-Tree 和 LSM-Tree，都是基于WAL-log，是一种只能 append 的二进制日志，leader 不仅将 wal-log 保存到磁盘，而且发送到 followers。

WAL 的 disadvantage 是太底层，wal 包含了磁盘那个位置的 block 需要修改哪些 bytes，这样耦合性比较高，如果format 改变了，基本上很难让 leader和follower 使用不同版本。

这个问题看起来更像是实现的问题，如果 replication 的协议支持follower 使用新的版本，就能完成 zore-downtime 的 upgrade。
首先更新 follower的版本，然后让leader failover，选择一个 follower 为 leader。如果replication 的协议不支持版本不一直，这就叫做 wal-shipping，会造成 upgrade-downtime。

3. Logical (row-based) log replication

这种方式就是 replication log 和具体的 storage device 解耦，使用不同格式。 relational database 的 logical log 一般都是一系列表示行的recorders。一个改变很多行的 statement 会
产生很多行这样的日志，后面有一个transaction 完成的标识符。这种方式是解耦的，所以可以容忍不同的版本，甚至不同的存储引擎。这也让外部的系统容易获得数据，
例如数仓、二级索引，外部缓存，这个技术叫做 change data capture。

4. trigger based replication

前面讲的都是数据库系统的，有时候我们需要更灵活的，例如你只想一部分数据，或者你想从一个数据库迁移到另一个数据库。这样你就应该将replication 提升到应用程序层面。
很多数据库提供了工具完成这个事情。许多关系型数据库提供了一项功能：triggers and stored procedures。
trigger 能让你注册用户代码，一旦数据发生改变，将会将数据放到另一给表格，外部系统可以访问。

### (5) Problems with Replication Lag

之前说了 Replication 是保证数据安全、忍受 node failures，实际上还可以提高 scalability。

Leader-based 需要所有的 write 请求都通过 master，将 read 分发到 slave，增加节点就能增加扩展性，
节点太多了就只能使用 异步的数据同步方式，否则可用性就很差。

然而一旦 slave 的数据 fallen behind 了，可能访问的数据过时了，这看起来问题不大，毕竟等等就行，这叫做  eventual consistency。

这个 eventually 是一个很NB的词，并没有告诉你具体时间，只是说最终会一致。我们称时间差为 the replication lag。这个 lag导致的问题我们接下来会讨论。

#### 1. Read your own writes

之前的 qq 空间有这个问题，当你发表了评论后页面会刷新，但是刷新后看不到自己发表的评论。
这时候你以为评论失败了，所以重新评论一下，然后刷新发现了自己评论了两次。

这种问题被叫做 “read-after-write consistency”，也叫做 “read-your-writes consistency”。有一些解决方案：

1. 如果是用户可以修改的信息，从 master 读。
2. 如果是一分钟之内修改过的数据，从 master 读，或者可以监控 lag 大于1分钟的follow 排除。
3. 客户端保存最近一次写的 timestamp，这样服务端保证读的时候先等 slave 的数据同步已经到了这个时间。这里的 timestamp 可以是逻辑时间。
4. 如果有多个数据中心，可能更复杂。

上面的解决方案只是一个 client，如果用户同事有多个 client 访问，例如手机和pc，这时候要考虑更多问题。

#### 2. Monotonic Reads

类似上面，如果连续两次访问来自于不同的 slave，可能出现时间倒退，例如看球赛的时候刚开始比分 1:1,刷新一下变成 1:0 了！

Monotonic Reads 就是保证上面的异常不发生的一致性。他比 eventually consistency 更强，比 strong consistency 更弱。

这种一致性的解决方案就是保证每个 userid 都从同一个 slave 读数据，例如根据 userid 的 hash。

#### 3. Consistent Prefix Reads

在一个群聊系统，如果你发现两个人的对话顺序反了，就是 Consistent Prefix Reads 没有得到保证。

这个问题解决方案需要保证有相互依赖的数据都按顺序写入。后续再  causal dependencies 和 happens-before 会讨论。

### (6). Solutions for Replication Lag

后续会讲解 事务 和 分布式事务。

## 2. Multi-Leader Replication

一个 leader 可能会有一些问题，例如写太多了可能会有性能问题。也有一些其他的选择， 例如多个 leader 。

### (1). “Use Cases for Multi-Leader Replication”

一般情况如果只有一个 datacenter 是不需要用多个 leader 的，所以在多个 datacenter 的时候会使用 multi-leader

#### Multi-datacenter operation

如果有多个 datacenter ，那么每个 datacenter 一个 leader 是一个不错的选择。
我们对比一下在 multi-datacenter 中使用 single-leader 和	multi-leader 的fare

1. Performance ，multi-datacenter 的性能可能好一些
2. Tolerance of datacenter outages ，multi-datacenter 不用担心leader 没了，所以更简单一点
3. Tolerance of network problems，由于 datacenter 之间的网络稳定性肯定不如 datacenter 内部，所以 multi-leader 更好点

总之，如果有多个 datacenter ，选择 multi-leader。尽管如此也会有缺点，
例如需要同时修改一份数据，可能发生冲突导致一边成功一边失败了，后面的 handle write conflict 会讨论。

同时，multi-leader 可能会有一些陷阱、和数据库的其他特性放一起经常出问题，例如主键自增。所以 multi-leader 需要尽量避免。

#### “Clients with offline operation”

以 印象笔记、icloud 为例，我们在手机、电脑 的笔记可以在线编辑，也可能离线编辑。
这个架构就和 Multi-Leader 类似，每个设备是一个 datacenter，服务端也是一个 datacenter，网络连接并不靠谱。

#### Collaborative editing

以 Google docs、wiki 为例，大家可以同事协同编辑，相关的算法就是 automatic conflict resolution。
你可能以为这个和 multi-leader 的 replication 不一样，实际上很类似。
一个人编辑的时候，change 马上就出现在 local replica ，然后是异步同步到服务器和其他的人。
如果你希望保证没有冲突，Application 需要在文档上面加锁，其余人获得锁之前不能编辑，这种就类似与 single-leader 的replica。
但是为了快速工作，你可能将锁分的很小，大家可以同时编辑，这就类似 multi-leader ，同时带来了 conflict resolution。

#### Handling Write Conflicts

如果两个人同时编辑，一个修改了标题一个修改了内容这不会有冲突，但是如果两个人都修改了标题该如何解决这个问题。
如果是 single-leader，并不会有这种问题，因为写都是有时间顺序的。后面的那个要么等待前面的修改事务成功或者失败，要么自己让让写任务失败。

- “Synchronous versus asynchronous conflict detection”

如果两个任务修改同一个数据都成功了，然后异步发现了冲突，这时候解决已经晚了，因为不知道放弃哪个。
当然原则上也可以通过同步修改的方式避免冲突，这样就和 single-leader 没什么区别了。

- Conflict avoidance

避免冲突是目前最好的方法了，例如用户可以修改自己信息的话，就让所有的修改都发送到同一个 datacenter 的 leader。

有时候也可能某个 datacenter 挂了或者用户去了另一个地方需要 datacenter 换了，需要使用别的方法避免冲突。这时候还是需要处理冲突。

- Converging toward a consistent state

对于 single-leader 的 writes 都有一个 sequential order，统一数据的最后一次写决定了数据。
对于 multi-leader 就不一样了，并不知道哪个 datacenter 的才是正确的，如果不进行修改就会出现各个 datacenter 不一致，
所以所有的更改必须 最终是 convergent ，也就是说 最终一致性。几种常见的 convergent 方法：

1. 简单粗暴，直接每个操作一个 uuid，例如 hash，或者时间戳，最大的那个胜出，如果使用时间戳，这种方法就叫做 Last-Write—wins（LLW）。
2. 给每个 replica 一个 id，id 最高的占主导地位，和上面的类似。
3. 合并数据，例如 一个标题是A，一个是B，最后就是 A|B
4. 用户自己实现冲突合并逻辑。

- Custom conflict resolution logic

自己实现冲突解决一般有两种方案：On Read 和 On Write ，相关细节比较简单

#### What is a conflict?

上面说的两个人同时修改标题的冲突很明显，所以可以通过一些程序控制，实际上也有一些不明显的的冲突。

例如两个人预定会议室，同一个会议室同一时间只能有一个人预定成功，但如果是 multi-leader 可能两个人都预订成功了。
或者注册的时候要求 username 不重复，但是有可能两个人同时申请同一个 username。

### (2) Multi-Leader Replication Topologies

Replication Topologies 是用来描述数据在节点间传播的路径。例如 circular 、star、all-2-all。

每一种都有优势和劣势。

## 3. Leaderless Replication

实际上 cassandra 采用了 leaderless 的架构，没有主节点。

### (1) Writing to the Database When a Node Is Down

leaderless 中，不存在 failover，因为没有主节点，因为一个节点在读数据的时候，并不需要从所有的节点都读数据，
二是给每个节点发送数据，然后每个节点都返回数据，根据 Version numbers 决定哪个是正确的。

#### Read repair and anti-entropy

系统需要保证每个 replica 上面的数据都是最终一致的，如果出现了节点挂掉然后恢复，如何保证它的数据是一致？

1. Read repair ，当用户读书节的时候，能够发现 stale 的值并且修复
2. Anti-entropy process 通过后台进程扫描数据发现不一致

#### Quorums for reading and writing

也就是上面所说的，读和写不需要全成功，只要保证 读成功数 r 和 写成功数 w 满足 r+w>n即可。这个叫做仲裁原理，类似抽屉原理，
n个抽屉，里面有 w 个抽屉放了一样的纸条，打开r个抽屉，想要看到纸条内容，只要 r + w >n。
一般情况下 r=w=(n+1)/2

- Limitations of Quorum Consistency

上面我们说了 r=w=(n+1)/2，但是如果你的数据库写的量特别小，读特别大，你也可以设置 r=n，w=1，总之只要覆盖即可。

如果你对一致性要求不高，也可以使用 w+r <= n，这样返回值可能 stale，不过这样的可用性更高。

然而尽管使 w+r > n，也会有一些其他的异常情况：

1. sloppy quorum 下无法保证
2. concurrently 写，和上一节类似，出现了写冲突？
3. 读和写 concurrently，这时候写只提交了一部分，其他的可能还未完全在 w 个节点成功
4. 如果写 成功不够w，但是这时候也没法 rollback，后续可能会读到失败的值，也可能读不到
5. 如果保存了 new value 的节点失败重启，从另一个保存了 old value 的节点修复数据，导致包含 new value 数据的节点少于 n
6. 还有极端情况，例如  “Linearizability and quorums”.

因此，即便是 达到了 quorum 的条件，也没法真正的实现一致性。
特别的，至于在 single-leader 中的 “reading your writes, monotonic reads, consistent prefix reads” 等弱一致性也没法得到保障。
所以前面说的异常可能也会发生，强一致性需要 consensus 或者 transaction 来保证。

#### Monitoring staleness

对于应用而言，是否返回最新版本的数据是很重要的。对于 leader-based 的设计，所有的问题都可以交给 replication-lag 来解决，
可以通过监控 replication-lag 监控系统
对于 leaderless 的设计，所有的数据并没有确定的顺序，监控很困难，而且如果没有 anti-entropy，数据可能过时很久都不修复。

### “Sloppy Quorums and Hinted Handoff”

简单介绍一下，例如 cassandra 中，保存三副本，读写都要成功 2 个 replica，我们写数据要成功 2个，
但是如果此时这三个节点挂了两个，只能写成功一个，可以先将另一份数据放到其他的节点上。
等对应的剩下的节点都恢复了，再将数据放回去，这个过程就是 hinted handoff。

### Multi-datacenter operation

这时候我们还是有N个副本，我们需要保证每在 每个 datacenter 都达到了 对应的 n/2 个副本。

### Detecting Concurrent Writes

类似之前的 handling write conflict，也是由于顺序不一样导致，所以也需要有保证 converge 的方法，
这样的方案也就是我们前面说的一样。但是我们这里讨论的更细致一点。

#### Last write wins (discarding concurrent writes)

对于不知道哪个 happens-first 的操作，我们称之为 concurrent，他们的 order 是不知道的。

尽管他们没有 order，我们可以人为给一个 order，例如 timestamp，也就是 Last write wins。

llw 解决了冲突问题，但是丢失了一些数据。所有使用 cassandra 的时候有人会给每个 key 后面加一个 uuid，这样能保证每个 key 多次写都不会丢失。

#### The “happens-before” relationship and concurrency

happens-before 的关系是决定是否并发的关键，以下情况下我们称两个操作是 concurrency： 
“neither happens before the other (neither knows about the other)”

因此，对于任何两个操作 A 和 B，只有两种可能，A happens-before B，B happens-before A，A 和 B 是 concurrency。

#### Capturing the happens-before relationship

上面已经知道两个操作只能有 3 种关系，我们如果发现 happens-before  的关系呢？其实有算法的。

首先假如 A 和 B两个人都在往同一个购物车添加商品，一开始 A添加了 商品1，B添加了商品 2，这两个是 concurrency的，
然后 A再添加 3，B添加4，这时候 这两个操作都是 3和4是 concurrency的，但是1和2 都是 happens-before 3 和4 的。

发现 concurrency 关系的算法如下：

1. server 的每个key有一个 version number，每次更新写这个 key 的时候，version number 就 +1并且跟新数据。
2. 当 client 读数据，返回所有没有 overwritten 的数据和最新的 version number。
3. client 写数据的时候，必须包含之前读到的那个 version number，并且合并所有的数据，
4. 当 server 收到带 version number 的写请求，覆盖所有小于等于 version number 的数据，保留大于 version number 的数据。

这样的算法就能发现所有的 happens-before 关系

#### Merging concurrently written values

类似前面的 conflict resolution，上面的算法需要 client 进行数据合并，合并算法大家自己查资料，例如 CRDT 。

#### Version vectors

上面的算法是 single-replica 情况，对于 multi-replica 的情况，需要每个副本一个 Version number，这被叫做 Version vectors。

## SUMmary

通过上面的讨论，我们明白了一句话，其实分布式 replica 最大的问题就是发现 happens-before 的依赖关系，也就是操作的顺序。

另外有大神说过，分布式只有两个问题， exactly-once 和 order。其中 exactly-once 后续会讨论，order 就是我们这里讨论最多的。




# 二、Partitioning

分区，also known as Sharding。相对其他的理论，这部分很简单了。主要就是跨分区的事务稍微复杂点。



# 三、Transactions

在一个数据系统中，很多东西可能出错，有可能是网络问题，硬件问题，系统 bug。为了可靠，我们必须让系统可靠，事务就是一个选择。
事务是我们设计应用的时候遵循的的一套规定，但实际上他并不是理所当然的，二是要通过设计底层算法来满足的，这次我们就看看这些算法。
这一节我们主要将重点放在单机的事务，下一节将会讨论分布式事务。

##  1. The Slippery Concept of a Transaction

现在几乎所有的 relational databases，以及一些 NoSql 都支持 transaction，但是 transaction 的定义到底是什么？
实际上一些 NoSql 摆脱了事务的约束，因为事务有时候严重影响了扩展性。都是一些  trade-offs

## 2. The Meaning of ACID

不同数据库的 ACID 实现方式是不同的，例如 isolation 的定义有很多，除了 ACID 还有一种约束叫做 BASE，就更加模糊了，一个一个看吧。

### (1) Atomicity

原子性其实有很多种定义，一般代表一个事物不可分，例如同时如果是多线程，另一个线程不可能看到转账中间的状态。

但是在事务中，并不是这个意思，上面的问题是 隔离性讨论范畴，Atomic 是说，例如一个转账的事务，分为两步，不可能只完成一步而另一步失败。

没有原子性保证，如果一个事物中途失败，我们没法知道哪些改变了哪些没改变。

### (2) Consistency

Consistency 是一个被严重重载的词，前面的 replica 中它是说主备不一致；CAP 中指的是 线性一致性，这里指的是一种 good state。

这个定义很模糊，因为他很难描述清楚。一是转账的时候，事务前后两个人的总金额要一样，
注册用户名的时候保证用户名不重复，不能违背外键约束，这都是一致性的问题。
AID 都是 properties of databases，数据库可能依赖 AID 来实现 Consistency，所以严格来说 Consistency 并不属于 ACID

### (3) Isolation

大部分数据都是多线程操作的，如果多个线程访问了同一条数据，就会出现类似前面讨论的 concurrency-write 问题，但是这里还有 read 的问题。
有的经典课本将它改为 serializability，意思是每个线程都可以假设只有他一个线程在操作数据库，数据库保证并发执行的结果和顺序执行的结果是一样的。
实际上真正的数据库很少会让操作只在一个线程里面，这样性能太低。

### (4) Durability

这个就是说事务一旦提交就持久化了，例如已经存入了 nonvolatile storage，例如 hard drive or SSD，结合 write-ahead log 等方式实现。
在 replicated databases 中，可能意味着已经复制到了好几个副本之中。
perfect durability 是不可能存在的，如果你的机房被恐怖分子炸掉了，硬盘都没了。

自从分布式理论有了以后， REPLICATION AND DURABILITY 就出现了一些变化，相关的可以查看资料。

## Single-Object and Multi-Object Operations

上面我们知道了，atomicity 和 isolation 描述了在客户端执行多个操作时候 ，数据库的做法。
前者表示如果事务进行到一半失败了必须删掉所有修改的数据，后者表示各个线程互不干扰。接下来看一个例子。

一个邮件系统需要显示未读邮件数量，如果每次用户登录都执行 count 操作会比较耗时，所以添加了一个变量 counter 表示未读邮件。

这时候每次收到一个邮件就 counter++ ，读了一份邮件就 counter --。
atomicity 能够保证 不会出现异常（例如已经没用邮件但是 counter >0）,不会出现 消息标记为已读但是 counter-- 执行失败。
如果标记已读的时候，突然收到一份邮件，这时候 counter 的 ++ 和 -- 是并发的，Isolation 保证不会出现异常。

Multi-object transactions 需要决定哪个读和写操作是同一个 事务。
对于 relational，就是同一个 tcp 连接的一个 start 和 commit 之间的操作属于同一个事务。
但是对于 Nosql，并没有相应的方法，尽管提供了对于的API，但是实际上不一定能保证事务。

### (1) Single-object writes

如果一个事务修改了一个 json 文件，但是由于网络问题，修改了一半。或者多个事务同时修改了一个字段，这时候也和上面一样需要进行单个对象事务控制。

一些数据库提供了 更复杂的 atomic 操作，例如 increment 可以避免 read-modify-write，类似的有 compare-and-set ，思想类似了乐观锁。
这样的操作实际上和 ACID 类似，compare-and-set 实际上也是 lightweight transactions，或者是市场化的 ACID。

### (2) The need for multi-object transactions

许多 NoSql 放弃了 multi-object transactions， 因为需要在不同的 partitions 上面实现事务。
我们真的需要 multi-object transactions 么？我们的应用能否只通过 single-object operations 实现。考虑以下情形：

1. 外键约束 关系型数据库可能有外键，我们执行一个操作的时候，需要考虑满足这些约束
2. document databases 很难 join，所以会有 denormalization，对应的修改必须是同时修改。
3. 考虑 secondary indexes，值修改了索引也要进行修改。

这些情形可以不用事务，但是容错就会相当复杂。

### (3) Handling errors and aborts

ACID 的主要目的是如果发生了错误，我可以完全丢弃掉这些改变，但是也有例外。
leaderless replication 在你的数据出现错误以后并不会进行修改。
如果发生了错误就直接 abort 然后重试也不是很好的选择：

1. 如果事务已经 succeeded，但是网络问题导致没有返回成功消息，你可能会标记为失败，然后重试会执行两次？
2. 如果集群 load 过高你这时候再次重试反而会是问题严重。
3. 有时候重试再多次也是失败的
4. 如果事物内部有一些操作，例如发邮件？你总不能每次重试就发一次邮件吧。

## 2. Weak Isolation Levels

上面说过 serializable 并不是最好的隔离方式，还有一些其他的方式。

### (1) Read Committed

Read Committed 有两个保证，1 是读数据的时候只能看到提交成功的数据，2 是写数据的时候只会覆盖写成功的数据。也就是没有脏读和脏写。

#### No dirty reads

如果你读了别的事务还没提交的数据就是脏读，如果覆盖了别的事务还没提交的数据就是脏写。
举个例子，如果你的账户余额为0，查看余额的时候，有人刚好在给你转账，
先给你加了 500，然后他的账户余额不够事务失败了，然后又 rollback，结果你刷新发现又变成了0。你看到 500 是脏数据，所以是脏读。
同理如果你给自己存钱的时候覆盖了这个 500，就是脏写。

Read Committed 隔离级别数据库需要保证不会出现脏读，也就是看到的数据是已经 commit 的。

#### No dirty writes

同样需要保证不会出现脏写，一般需要加锁，等待前一个事务 commit 或者 abort

#### Implementing read committed

简单粗暴的方式就是加锁，这也是大部分数据库 保证 没有 No dirty writes 的方式，但是脏读也加锁就会降低性能。
所以脏读一般是通过保存两份数据，一个是未提交的数据，一个是已经提交的数据。读的时候返回已经提交的数据。

### (2) Snapshot Isolation and Repeatable Read

貌似 上面已经很完美了，但是看看下面的例子：
还是两个人转账，A 把 500 块转给 B，首先把 A 设置为 0，然后 B 设置为 500。
此时B 查看了两个用户的账户，在还没开始的时候查看了A的，结束的时候查看了B的，发现两个人都是 0.

上面的问题和之前的反了过来。脏读是读了修改到一半的数据，不可重复读是修改了读到一半的数据。
这貌似不是一个大问题，只要等一会儿再读，就发现是正确的。但是注意以下情况：

1. Backups 如果我再备份数据，类似这里的读。
2. Analytic queries 

Snapshot isolation 就是解决这个问题的方法。实现 快照隔离需要用 MVCC 算法。
之前每条数据都有两个版本，未提交的和提交的，现在改成多个，也就是每一条保存多个版本的数据，算法如下：

1. 每个事务都有一个 txid。
2. 每一行数据有一个 created_by 段包含了 tx_id。
3. 每一行数据有一个 deleted_by 字段默认空值，如果数据删除了，实际上不会删的，二是 deleted_by 会有一个 txid 标志它被删掉。
4. 如果确定没有实物能访问到被删掉的数据，就真的删了被删掉的数据。
5. update 等于 delete + create

读数据时候的 Visibility rules 如下：

1. 每个 transaction 开始的时候会将所有正在执行的 transactions 列出，这些事务的修改无论成功与否都 ignore。
2. 所有 abort 的事务做的修改 都 ignore。
3. 比它大的 txid 做出的修改都 ignore。
4. 其他的修改都是 可见的。
5. 这里的修改指的是 create 和 delete

换个方式，只有下面的情况才是 visible：

1. transaction 开始的时候，已经 commit 的数据
2. 别的事物虽然已经删了数据，但是还没有提交的。

### (3) Indexes and snapshot isolation

在 multi-version 的 databases 里面 index 如何工作? 可以和数据一样，有多个索引，但也可以在有多个版本的时候避免索引更新。
还有的结构是 appendOnly 的结构。。。。

### (4) Repeatable read and naming confusion

Repeatable read 其实一开始是没有的，也是慢慢被发现的，所以它的名字有很多，实现方式也不一样，提供的一致性保证也不一定一样。

## 3. Preventing Lost Updates

前面只讨论了 dirty writes，类似我们讨论的并发写，还有一些问题我们没有讨论。最有名的就是  Lost Updates，例如并发给某个数据 +1.
一般可能出现的情况：
1. 查询数据并增加，例如：转账的时候同时转账。
2. 同时编辑某个 wiki 文档的一个部分。
3. 修改一个 json 字符串

### (1) Atomic write operations

大部分 relational db 都支持 atomic 写，即 `UPDATE counters SET value = value + 1 WHERE key = 'foo';` 而不需要 read-modify-write。

一般情况下都是通过 exclusive lock 方式实现，这种方式也被称为  cursor stability，另一种就是强制要求所有的操作在一个线程里。

但是有些 orm 框架让你很容易写出 unsafe 的并发更新bug而且很难查出来。

### (2) Explicit locking

```
SELECT * FROM figures
  WHERE name = 'robot' AND game_id = 222
  FOR UPDATE;
```

这样直接锁住记录

### (3) Automatically detecting lost updates

加锁的方式是让操作序列化执行，同样可以让每个并发执行，如果发生了错误就 abort 重试。
Lost update detection 是一种很高级的特性。

### (4) Compare-and-set

这种方式就是写之前先读，如果数据没有变化再写。
但是如果写之前读的是还没提交的数据，可能还是有问题的。

### (5) Conflict resolution and replication

如果有多个副本，写数据都是并发，同步数据是异步的，这时候 locks 和 compare-and-set 的方式以及没法保证数据一致性了。

## 4. Write Skew and Phantoms

除了上面的问题，可能还有新的问题。看下嘛的例子：
我们有个医生门诊应用，每个医生可以请假，但是至少保证有一个医生没有请假。医生请假的时候会查询目前在岗的医生，发现有人就可以请假。
但是如果有一个时间所有的医生都同时查看，发现大家都在，然后大家都一起请假，就违背了约束。



### (1) Characterizing write skew

这种问题叫做 write skew，既不是脏写也不是 lost update。对应的读问题叫做 phantoms read。
因为两个事务更新的是两个数据（每个医生设置自己的状态为请假），这里的冲突不太明显，但是确实违背了约束。

write skew 可以看出广义的 lost update，write skew 是多个事务读了同一个数据然后更新了其中的部分数据。
如果更新的是同一条数据就会得到 dirty write 或者 lost update。

lost update 我们有很多种方案可以解决，但是貌似这里就不行了：
1. Atomic single-object 没法解决，因为 这里有多个 obj
2. automatic detection 很难，因为这种问题需要 Serializable 的隔离级别
3. 通过添加唯一位数、主键约束都不一定可以，例如这里有多个医生的在岗状态，如果加约束需要每次都扫描全表。

还有很多 write skew 的问题，
会议室预约系统如何保证一个会议室在同一时间不会被两个人预定？
Multiplayer game 怎么保证两个用户不会出现在同一个点，
用户名唯一性约束等

### (2) Phantoms causing write skew

上面的问题有一个共同点：
1. 查询是否满足多个条件
2. 根据上面的结果做出更新
3. 第二步的更新导致第一步的条件不成立

注意一定是有更改，如果是只读的， snapshot isolation 就能保证不会出现异常。

### (3) Materializing conflicts

貌似有的问题是我们没法加锁，例如第一个 不知道给哪个医生加锁？但是如果能够物化冲突，找到加锁的地方就能够解决了。

会议室预定的，我们可以给每个会议室每个时间段创建一条记录，然后加锁。问题在于有的我们很难物化，例如用户名唯一性，
不可能给每个用户名都创建一个索引。大多数还是使用 Serializability 的隔离级别

## 4. Serializability

这是目前 strongest isolation level，让数据库表现的类似所有的事务都 execute in parallel。

如何实现  implementing 并且它的效率如何是需要考虑的问题。目前三种实现：顺序执行、2PL、SSI

### (1) Actual Serial Execution


#### Encapsulating transactions in stored procedures

### (2) Two-Phase Locking (2PL)

很久一段时间，实现 serial 只有一种方法，那就是 2PL，注意这里的 2pl 和 后面的 2pc 不一样。

我们知道锁的作用就是更新时候保证一个是在另一个完成后进行的。2PL类似，很多个线程都可以同时读一条数据，
但是如果想要更改数据，就需要使用 exclusive access ：
1. A 如果读了数据 然后 B 想更改这条数据，B 需要等待 A 事务 commit 或者 abort 
2. A 如果修改了数据 然后 B 想读这条数据，B 需要等待 A 事务 commit 或者 abort 

2PL 和 前面的锁不一样的地方在于，writers 不仅会锁住其他的 writers，还会锁住其他的 readers，反过来也一样。
这就是和 snapshot isolation 不一样的地方，所以他能够防止前面所有的错误。

#### Implementation of two-phase locking

实现 2PL 的方式就是给每个记录加锁，锁可以是 shared 或者 exclusive。使用方式如下：
1. 如果线程 A 想读 一个记录，需要获得 shared mode 的锁。所谓 shared mode 就是多个 shared mode 的线程可以共用锁
2. 如果线程 A 想写 一个记录，需要获得 exclusive mode 的锁。所谓 exclusive mode，就是只能一个人获得锁
3. 如果一个事务先 读然后写，需要将对应的 shared mode 的锁 升级为 exclusive mode 的锁。
4. 一个事务获得了锁，就需要一个 hold 住，知道结束（commit或者 abort）.

这就是 two-phase 的由来，phase1 就是 acquired lock(execute transaction), phase2 就是 release locks(end of the transaction)
由于使用了很多 locks，很容易出现 A 等待 B释放锁，同时 B 等待 A，这就是 死锁，数据库一般可以自动发现死锁。

#### Performance of two-phase locking

一直依赖不用 2PC 的原因就是性能太差了，一方面获得锁释放锁有性能开销，更重要的还是并发度太低。
而且一旦出现事务等待，不知道要等待多久。另外死锁出现的概率大了很多。

#### Predicate locks

上面的预定会议室程序，如果一个用户预定了一个某个时间段的某个地点，另一个用户此时还可以预约，只要不是相同的时间和地点。

这时候我们可以使用 Predicate locks，和 2PL 类似，但是不一样的是 2PL 对查询的某一条数据加锁， 
Predicate locks 是加在所有的符合某个条件的所有数据。

这里的关键在于 Predicate locks 不仅可以加在目前存在的记录上面，还可以加在未来会写进来的数据上。
如果 2PL 包含了 Predicate locks，就可以防止任何形式的 phantom，也就成了 Serializable。

#### Index-range locks

Predicate locks 并不好用，如果有太多 lock，判断每个lock 很耗时，所以很多 2PL 使用的是 index-range locking。

例如我们要锁住 某个会议室在 明天下午三点 ，这判断起来很麻烦，我们可以直接将这个会议室锁起来！或者把所有的会议室 下午三点都锁起来。
由于 room_id 或者 time 可能有索引，所以判断起来会比较快。

index-range locks 并没有 Predicate locks 精确，但是可以降低查询量，也是一个好的折中。
如果没有索引，就锁住整个表格，这样就会降低性能，但是也是一个折中。

### Serializable Snapshot Isolation (SSI)

前面我们发现，提供的隔离性越好，性能就越差？貌似性能和隔离性一定是 odds？不一定，有个 SSI 算法比较新，提供了很好的性能。

#### Pessimistic versus optimistic concurrency control

2PL 是 pessimistic 悲观锁，如果可能会出错那么就等待安全了再操作，类似 mutual exclusion。
Serial execution ，就是极端的 pessimistic。

serializable snapshot isolation 是一种 optimistic concurrency control technique. 
先假定不会出错，然后 commit 的时候判断有没有出错。

### Decisions based on an outdated premise

我们前面出现的 write skew 的时候，都有一个通用的模式：
一个 transaction 读一些数据，检查结果并且做一些动作，但是提交数据的时候，这些数据可能已经被另一个事务更改。

transaction 会基于 premise 做出一些 action，等后面准备 commit 的时候，这个 premise 已经不正确了。
换句话说，读的数据和写的数据之间有一个 causal dependency。如何发现 outdated premise，主要有两种：

1. Detecting reads of a stale MVCC object version (uncommitted write occurred before the read)
2. Detecting writes that affect prior reads (the write occurs after the read)

#### Detecting stale MVCC reads 

在 snapshot isolation 中使用了 multi-version 的数据，每个事务不会读到当前还没有提交的数据，
所以其他的线程可能修改了这个数据，为了防着这种情况发生，db 需要记录由于 MVCC visibility rules 而忽略的数据，
当这个事务 commit 的时候，检查这些数据是否提交，如果提交了就 abort。

为何要等待 commit 的时候判断而不是直接放弃？因为 当前事务可能是只读，或者另一个事务会失败，或者另一个事务会持续很久。

#### Detecting writes that affect prior reads

当一个 transaction 写数据的时候，他需要查看最近读了这个数据的事务，通知这些事务他们的他们的数据已经过时了。

#### Performance of serializable snapshot isolation

和 2PL 相比，SSI 不用阻塞线程。读写之间是不会阻塞，导致延迟低。

aborts rate 对性能影响很大。但是 SSI 对 长事务 的敏感程度 肯定比 2PL 和 real serial 低。




# 第五部分、案例

https://medium.freecodecamp.org/how-to-system-design-dda63ed27e26

http://blog.gainlo.co/index.php/2015/10/22/8-things-you-need-to-know-before-system-design-interviews/

https://engineering.videoblocks.com/web-architecture-101-a3224e126947

# 第六部分、资料

es设计架构，良心参考资料：
https://blog.insightdatascience.com/anatomy-of-an-elasticsearch-cluster-part-i-7ac9a13b05db
https://blog.insightdatascience.com/anatomy-of-an-elasticsearch-cluster-part-ii-6db4e821b571
https://blog.insightdatascience.com/anatomy-of-an-elasticsearch-cluster-part-iii-8bb6ac84488d

lsm b+tree 资料
https://medium.com/databasss/on-disk-io-part-3-lsm-trees-8b2da218496f
