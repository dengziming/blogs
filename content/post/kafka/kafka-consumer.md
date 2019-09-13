---
date: 2019-04-20
title: "kafka-consumer"
author: "邓子明"
tags:
    - kafka
    - 源码
categories:
    - kafka源码
comment: true
---




## KafkaConsumer


## 传递保证语义

exactly once 是分布式系统两大难题之一。kafka 也没有完全解决，需要自己结合业务情况。

## Consumer Group Rebalance

同一个 topic 不同的分区会分配给 一个 group 中的不同消费者，怎么分配是一个复杂的算法。我们写代码很少自己实现，
实际上 spark 如果是多个 task 会启动多个消费线程，这些线程分配 spark 帮我们实现好了，接下来我们看看相关代码。

方案一：
zookeeper 保存了 /consumers/[group_id]/ids，记录属于此Consumer Group的消费者的Id。
zookeeper 保存了 /consumers/[group_id]/owners，记录属于此分区与消费者的对应关系。
zookeeper 保存了 /consumers/[group_id]/offsets，记录此Consumer Group在某个分区上面的消费位置。
broker 也会有相应的保存。每次 broker 和 consumer 发生变化，消费者都可以订阅 zookeeper。
但是验证依赖 zookeeper 有问题。

方案二：
服务端用 GroupCoordinator 管理 Consumer Group。
消费者加入 Consumer Group 时，向任一 Broker 发送 ConsumerMetadataRequest 消息，消息包含了 groupId，
ConsumerMetadataResponse 作为响应，包含了管理 对应的 Consumer Group 的 GroupCoordinator 信息。
消费者可以根据这个 GroupCoordinator 信息，连接到 GroupCoordinator 并周期性 HeartbeatRequest，
出现 timeout 就可以重新请求。出现 reblance 可以发送 JoinGroupRequest，加入 Consumer Group。

总之有两种操作，一个是 ConsumerMetadataRequest ，一个是 JoinGroupRequest，但是分区策略依赖了 服务端不灵活。

方案三：

GroupCoordinator 管理 Consumer Group，分区分配 由消费者处理。消费者中有一个是 Leader，leader 进行分区分配，
发送给 GroupCoordinator，GroupCoordinator 再返回给每个 consumer 线程对应的 topic partition。

## KafkaConsumer 分析

关键字段：

```java
PRODUCER_CLIENT_ID_SEQUENCE：clientId的生成器，如果没有明确指定client的Id，则使用字段生成一个ID。
clientId：Consumer的唯一标示。
coordinator：控制着Consumer与服务端GroupCoordinator之间的通信逻辑，读者可以将其理解成Consumer与服务端GroupCoordinator通信的门面。
keyDeserializer和valueDeserializer：key反序列化器和value反序列化器，参见第2章相关章节。
fetcher：负责从服务端获取消息。
interceptors：ConsumerInterceptor集合，ConsumerInterceptor.onConsumer()方法可以在消息通过poll()方法返回给用户之前对其进行拦截
或修改；ConsumerInterceptor. onCommit()方法也可以在服务端返回提交offset成功的响应时对其进行拦截或修改。与前面介绍的ProducerInterceptors类似。
client：负责消费者与Kafka服务端的网络通信。
subscriptions：维护了消费者的消费状态。
metadata：记录了整个Kafka集群的元信息。
currentThread和refcount：分别记录了当前使用KafkaConsumer的线程Id和重入次数，KafkaConsumer的acquire()方法和release()方法实现了一个“轻量级锁”，它并非真正的锁，仅是检测是否有多线程并发操作KafkaConsumer而已。
```

关键方法：

```java
subscribe()方法：订阅指定的Topic，并为消费者自动分配分区。
assign()方法：用户手动订阅指定的Topic，并且指定消费的分区。此方法与subscribe()方法互斥，在后面会详细介绍是如何实现互斥的。
commit*()方法：提交消费者已经消费完成的offset。
seek*()方法：指定消费者起始消费的位置。
poll()方法：负责从服务端获取消息。
pause()、resume()方法：暂停/继续Consumer，暂停后poll()方法会返回空。
```

### ConsumerNetworkClient

类似 Producer 中的 NetworkClient，ConsumerNetworkClient 进行了封装，方法更复杂。

```java
 private boolean trySend(long now) {
     // send any requests that can be sent now
     boolean requestsSent = false;
     // 遍历 unsent 中的数据
     for (Map.Entry<Node, List<ClientRequest>> requestEntry: unsent.entrySet()) {
         Node node = requestEntry.getKey();
         Iterator<ClientRequest> iterator = requestEntry.getValue().iterator();
         while (iterator.hasNext()) {
             ClientRequest request = iterator.next();
             // 判断是否能发送
             if (client.ready(node, now)) {
             	// 放入 InFlightRequests 队列中。
                 client.send(request, now);
                 iterator.remove();
                 requestsSent = true;
             }
         }
     }
     return requestsSent;

```

clientPoll 方法 ，调用 NetworkClient.poll()

send 方法，将 ClientRequest 放到 unsent 中等待发送。


