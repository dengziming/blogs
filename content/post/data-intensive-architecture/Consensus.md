---
date: 2019-04-15
title: "分布式存储-Consensus"
author: "邓子明"
tags:
    - 架构
    - 大数据
categories:
    - 设计高并发架构
comment: true
---



# 七、Consensus

之前介绍了 replica、partition、transaction。本次我们开始分布式系统相关讨论。

##  1. Consistency Guarantees

分布式系统会有一个比较模糊的概念，eventual consistency,这是一个 weak guarantee，还会学习一些 stronger consistency，
然后我们又回到之前谈论过的 order，最后讨论分布式事务。

## 2. Linearizability

在 eventually consistent 中，我们保证数据最终一致，这很 confusion，能不能保证任何时候访问都是一样的，也就是数据好像只有一个 replica 一样？
这就是 linearizability（也叫“atomic consistency [7], strong consistency, immediate consistency, external consistency [8]”）

在 linearizable 中，client 写成功以后，其他的 clients 都必须马上能看到这个数据，保证数据都是最新的、不过时的数据，
换句话说 linearizable 就是 recency guarantee，我们可以看一些 not linearizable 的 例子：

两个人通过手机视频看世界杯，但是由于访问了不同的 replica，存在 replica-lag，导致数据不一致，
其中至少有一个看到的是 stale value，也就是 violation of linearizability。

### (1) What Makes a System Linearizable?

上面已经说了，Linearizable 其实就是说多个 replica 表现的像分一个 replica 一样。
在 distributed systems 的书面语中，数据被称为 register，可以是一个 redis 的一个 key，mysql 的一行数据。
register 我们定义两种基本类型的 operation：
1. read(x) ⇒ v means the client requested to read the value of register x, and the database returned the value v.
2. write(x, v) ⇒ r means the client requested to set the register x to value v, and the database returned response r (which could be ok or error).”

有这些还不能完全定义 linearizability：如果读写并发，可能读到 new value 或者 old value，会出现值 back and forth 的情况。
这显然不能满足需求，所以我们需要加更多 constraint，我们要求如果 A 某个时刻读到了 new value，B再次读不能出现 old value。
这样我们就可以在定义一种类型的 operation：
3. cas(x, vold, vnew) ⇒ r means the client requested an atomic compare-and-set operation (see “Compare-and-set”). If the current value of the register x equals vold, it should be atomically set to vnew. If x ≠ vold then the operation should leave the register unchanged and return an error. r is the database’s response (ok or error).

linearizability 需要 所有的 operations 都 “move forward in time”，也就是前面的 recency guarantee，
一旦 new value 读或者写了，后续的都是 new value。

一些有趣的要点：
1. 

### (2) Relying on Linearizability

通过 Linearizability 能实现很多功能，

### (3) Implementing Linearizable Systems

实现 Linearizable 最简单的方式就是真的只保存一份数据，但是我们肯定希望是分布式的。常见几种方式：

1. Single-leader replication (potentially linearizable) 一个 leader 能基本保证，但是设计上或者并发的bug可能出现问题。另外单节点 failover 问题。
2. Consensus algorithms (linearizable) 
3. Multi-leader replication (not linearizable)
4. Leaderless replication (probably not linearizable)

#### Linearizability and quorums

很多网络延迟导致 quorums 是没法保证 Linearizability 的。

#### The Cost of Linearizability

#### The CAP theorem

CAP 理论的 C 指的就是这个 Linearizability，如果要想线性一致，就没法满足可用性。


