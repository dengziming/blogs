---
date: 2018-12-31
title: "分布式存储-Transactions"
author: "邓子明"
tags:
    - 架构
    - 大数据
categories:
    - 设计高并发架构
comment: true
---



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

