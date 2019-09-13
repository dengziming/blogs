---
date: 2018-12-31
title: "分布式存储-replication"
author: "邓子明"
tags:
    - 架构
    - 大数据
categories:
    - 设计高并发架构
comment: true
---



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

