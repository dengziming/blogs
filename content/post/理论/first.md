---
date: 2018-05-01
title: "分布式算法理论"
author: "邓子明"
tags:
    - 理论
    - 分布式
categories:
    - 分布式理论
comment: true
---


## 一、算法



## 二、理论

### 1. Two-phase commit protocol

直接翻译维基百科的解释了：
https://en.wikipedia.org/wiki/Two-phase_commit_protocol

In transaction processing, databases, and computer networking, the two-phase commit protocol (2PC) is a type of atomic commitment protocol (ACP). 

在处理事务、数据库和计算机网络，2PC是一种原子性提交协议

It is a distributed algorithm that coordinates all the processes that participate in a distributed atomic transaction on whether to commit or abort (roll back) 
the transaction (it is a specialized type of consensus protocol). 

2PC是一种 协调所有参与分布式原子事务的进程 是否 提交或者放弃提交事务 的分布式算法。

The protocol achieves its goal even in many cases of temporary system failure (involving either process, network node, communication, etc. failures), 
and is thus widely used.[1][2][3] However, it is not resilient to all possible failure configurations, 
and in rare cases, user (e.g., a system's administrator) intervention is needed to remedy an outcome. 

这个协议在很多临时的系统系统失败的时候达到它的目的，所以会广泛应用。但是，这个挺不是在所有可能的配置都是可伸缩的。
在极少数情况下，需要用户人为干预补救结果。

To accommodate recovery from failure (automatic in most cases) the protocol's participants use logging of the protocol's states. 

为了容纳失败后的恢复（大多时候是自动的），协议的参与者通过日志记录协议的状态。

Log records, which are typically slow to generate but survive failures, are used by the protocol's recovery procedures. 

日志记录，一般用在协议的恢复过程中，一般生成比较慢，但是失败的时候不会被删掉。

Many protocol variants exist that primarily differ in logging strategies and recovery mechanisms. 

这个协议的有很多延伸算法，不同点主要是记录日志的策略和恢复机制。

Though usually intended to be used infrequently, recovery procedures compose a substantial portion of the protocol, 

尽管很少被使用，恢复策略是这个协议的关键部分，

due to many possible failure scenarios to be considered and supported by the protocol.

由于许多可能的恢复场景都需要考虑支持这个协议。

In a "normal execution" of any single distributed transaction ( i.e., when no failure occurs, which is typically the most frequent situation), the protocol consists of two phases:

在很多单一的分布式系统事务中，一个简单普通操作，这个协议包含两部分：

1. The commit-request phase (or voting phase), in which a coordinator process attempts to prepare all the transaction's participating processes (named participants, cohorts, or workers) 
to take the necessary steps for either committing or aborting the transaction and to vote, either "Yes": commit (if the transaction participant's local portion execution has ended properly), or "No": abort (if a problem has been detected with the local portion), 

第一个是请求提交阶段，或者投票阶段，这个阶段协调者尝试 准备所有的事务的参与者（我们也叫他们participants, cohorts, or workers）采取必要的步骤提交或者放弃，然后投票yes或者no，分别代表commit或者abort

2. and The commit phase, in which, based on voting of the cohorts, the coordinator decides whether to commit (only if all have voted "Yes") or abort the transaction (otherwise), 
and notifies the result to all the cohorts. The cohorts then follow with the needed actions (commit or abort) with their local transactional resources (also called recoverable resources; e.g., database data) 
and their respective portions in the transaction's other output (if applicable).

第二个阶段是提交截断，基于 cohorts 的投票结果，coordinator决定是否提交或者放弃。然后向 所有的 cohorts 通知结果。
cohorts将会使用本地资源(also called recoverable resources; e.g., database data) 和他们各自比例 来执行对应的actions 。

Note that the two-phase commit (2PC) protocol should not be confused with the two-phase locking (2PL) protocol, a concurrency control protocol.


#### Assumptions

The protocol works in the following manner: one node is a designated coordinator, 
which is the master site, and the rest of the nodes in the network are designated the cohorts. 

该协议以如下方式工作：一个节点是指定的协调器，它是主站点，而网络中的其余节点被指定为同伙。

The protocol assumes that there is stable storage at each node with a write-ahead log, that no node crashes forever, 
that the data in the write-ahead log is never lost or corrupted in a crash, and that any two nodes can communicate with each other. 

该协议假定在每个节点上有一个提前写入日志的稳定存储，即没有节点永远崩溃，写入前日志中的数据在崩溃中从未丢失或损坏，并且任何两个节点可以彼此通信。

The last assumption is not too restrictive, as network communication can typically be rerouted. 
The first two assumptions are much stronger; if a node is totally destroyed then data can be lost.


最后一个假设不是太严格，因为网络通信通常可以重新路由。前两个假设强得多；如果一个节点被完全破坏，那么数据就会丢失。

The protocol is initiated by the coordinator after the last step of the transaction has been reached. 
The cohorts then respond with an agreement message or an abort message depending on whether the transaction has been processed successfully at the cohort.

该协议是在事务的最后一步到达之后由协调器发起的。同伙随后根据协议消息或中止消息来响应，这取决于事务是否在队列中被成功处理。

#### Basic algorithm

1. Commit request phase or voting phase

The coordinator sends a query to commit message to all cohorts and waits until it has received a reply from all cohorts.

coordinator 给所有的 cohorts 发送一个commit message 然后等到所有的 cohorts 回复。

The cohorts execute the transaction up to the point where they will be asked to commit. 
They each write an entry to their undo log and an entry to their redo log.

cohorts执行事务，那是他们将会被要求提交，他们各自写一个条目到他们的撤销日志和一个条目到他们的重做日志。

Each cohort replies with an agreement message (cohort votes Yes to commit), if the cohort's actions succeeded, 
or an abort message (cohort votes No, not to commit), if the cohort experiences a failure that will make it impossible to commit.

每个 cohort 回复一个 agreement message ，如果这个cohort的action成功（cohort votes Yes to commit），
或者 cohort experiences a failure that will make it impossible to commit，回复 an abort message (cohort votes No, not to commit)

2. Commit phase or Completion phase

- Success：If the coordinator received an agreement message from all cohorts during the commit-request phase:

如果 coordinator 收到从所有的 cohorts 收到一个an agreement message

The coordinator sends a commit message to all the cohorts.

coordinator 给所有的 cohorts 发送一个 commit message

Each cohort completes the operation, and releases all the locks and resources held during the transaction.

每个 cohort 完成 operation，释放 transaction 持有的 locks and resources

Each cohort sends an acknowledgment to the coordinator.

每个 cohort get coordinator 发送一个 acknowledgment。

The coordinator completes the transaction when all acknowledgments have been received.

coordinator 收到所有的 acknowledgments 完成 transaction 。

- Failure：If any cohort votes No during the commit-request phase (or the coordinator's timeout expires):

The coordinator sends a rollback message to all the cohorts.

Each cohort undoes the transaction using the undo log, and releases the resources and locks held during the transaction.

Each cohort sends an acknowledgement to the coordinator.

The coordinator undoes the transaction when all acknowledgements have been received.

类似上面的过程。

3. Message flow

```
Coordinator                                         Cohort
                              QUERY TO COMMIT
                -------------------------------->
                              VOTE YES/NO           prepare*/abort*
                <-------------------------------
commit*/abort*                COMMIT/ROLLBACK
                -------------------------------->
                              ACKNOWLEDGMENT        commit*/abort*
                <--------------------------------  
end
```

An * next to the record type means that the record is forced to stable storage.[4]

`*` 代表 record 已经强制刷新到 stable storage。

## Disadvantages

The greatest disadvantage of the two-phase commit protocol is that it is a blocking protocol. 
If the coordinator fails permanently, some cohorts will never resolve their transactions: After a cohort has sent an agreement message to the coordinator, it will block until a commit or rollback is received.

是一个 阻塞式协议，coordinator 如果失败了，cohorts可能会永远得不到回复。

上面介绍了 2PC，2PC的劣势已经了解，接下来我们认识3PC

### Three-phase commit protocol

```
status Coordinator                              Cohort status
                              can COMMIT ?
                -------------------------------->
                              VOTE YES/NO           Uncertain
                <-------------------------------    timeout cause abort
commit authorized  
                              precommit
                -------------------------------->    prepare to commit
                              ACKNOWLEDGMENT        
                <--------------------------------  
finalize commit               do COMMIT
timeout cause abort -------------------------------->
                              have COMMITED           commited
                <--------------------------------  
end
```

