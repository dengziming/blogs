---
date: 2019-04-18
title: "kafka-producer"
author: "邓子明"
tags:
    - kafka
    - 源码
categories:
    - kafka源码
comment: true
---


## 构造函数

```scala
private KafkaProducer(ProducerConfig config, Serializer<K> keySerializer, Serializer<V> valueSerializer) {
    try {
        //省略一段
        
        this.metrics = new Metrics(metricConfig, reporters, time);
        // 构造各个组件
        this.partitioner = config.getConfiguredInstance(ProducerConfig.PARTITIONER_CLASS_CONFIG, Partitioner.class);
        long retryBackoffMs = config.getLong(ProducerConfig.RETRY_BACKOFF_MS_CONFIG);
        
        this.metadata = new Metadata(retryBackoffMs, config.getLong(ProducerConfig.METADATA_MAX_AGE_CONFIG));
        this.maxRequestSize = config.getInt(ProducerConfig.MAX_REQUEST_SIZE_CONFIG);
        this.totalMemorySize = config.getLong(ProducerConfig.BUFFER_MEMORY_CONFIG);
        
        this.compressionType = CompressionType.forName(config.getString(ProducerConfig.COMPRESSION_TYPE_CONFIG));
        
        
        this.accumulator = new RecordAccumulator(config.getInt(ProducerConfig.BATCH_SIZE_CONFIG),
                this.totalMemorySize,
                this.compressionType,
                config.getLong(ProducerConfig.LINGER_MS_CONFIG),
                retryBackoffMs,
                metrics,
                time);
        List<InetSocketAddress> addresses = ClientUtils.parseAndValidateAddresses(config.getList(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG));
        this.metadata.update(Cluster.bootstrap(addresses), time.milliseconds());
        ChannelBuilder channelBuilder = ClientUtils.createChannelBuilder(config.values());
        
        NetworkClient client = new NetworkClient(
                new Selector(config.getLong(ProducerConfig.CONNECTIONS_MAX_IDLE_MS_CONFIG), this.metrics, time, "producer", channelBuilder),
                this.metadata,
                clientId,
                config.getInt(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION),
                config.getLong(ProducerConfig.RECONNECT_BACKOFF_MS_CONFIG),
                config.getInt(ProducerConfig.SEND_BUFFER_CONFIG),
                config.getInt(ProducerConfig.RECEIVE_BUFFER_CONFIG),
                this.requestTimeoutMs, time);
        this.sender = new Sender(client,
                this.metadata,
                this.accumulator,
                config.getInt(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION) == 1,
                config.getInt(ProducerConfig.MAX_REQUEST_SIZE_CONFIG),
                (short) parseAcks(config.getString(ProducerConfig.ACKS_CONFIG)),
                config.getInt(ProducerConfig.RETRIES_CONFIG),
                this.metrics,
                new SystemTime(),
                clientId,
                this.requestTimeoutMs);
        String ioThreadName = "kafka-producer-network-thread" + (clientId.length() > 0 ? " | " + clientId : "");
        
        // 创建启动现场
        this.ioThread = new KafkaThread(ioThreadName, this.sender, true);
        this.ioThread.start();

        this.errors = this.metrics.sensor("errors");
        // ....
}
```

## send 方法

```scala
@Override
public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback) {
    // intercept the record, which can be potentially modified; this method does not throw exceptions
    ProducerRecord<K, V> interceptedRecord = this.interceptors == null ? record : this.interceptors.onSend(record);
    return doSend(interceptedRecord, callback);
}
```

doSend 方法
```scala
private Future<RecordMetadata> doSend(ProducerRecord<K, V> record, Callback callback) {
    TopicPartition tp = null;
    try {
        // first make sure the metadata for the topic is available
        long waitedOnMetadataMs = waitOnMetadata(record.topic(), this.maxBlockTimeMs);
        long remainingWaitMs = Math.max(0, this.maxBlockTimeMs - waitedOnMetadataMs);
        byte[] serializedKey;
        try {
            serializedKey = keySerializer.serialize(record.topic(), record.key());
        } catch (ClassCastException cce) {
            throw new SerializationException("Can't convert key of class " + record.key().getClass().getName() +
                    " to class " + producerConfig.getClass(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG).getName() +
                    " specified in key.serializer");
        }
        byte[] serializedValue;
        try {
            serializedValue = valueSerializer.serialize(record.topic(), record.value());
        } catch (ClassCastException cce) {
            throw new SerializationException("Can't convert value of class " + record.value().getClass().getName() +
                    " to class " + producerConfig.getClass(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG).getName() +
                    " specified in value.serializer");
        }
        int partition = partition(record, serializedKey, serializedValue, metadata.fetch());
        int serializedSize = Records.LOG_OVERHEAD + Record.recordSize(serializedKey, serializedValue);
        ensureValidRecordSize(serializedSize);
        tp = new TopicPartition(record.topic(), partition);
        long timestamp = record.timestamp() == null ? time.milliseconds() : record.timestamp();
        log.trace("Sending record {} with callback {} to topic {} partition {}", record, callback, record.topic(), partition);
        // producer callback will make sure to call both 'callback' and interceptor callback
        Callback interceptCallback = this.interceptors == null ? callback : new InterceptorCallback<>(callback, this.interceptors, tp);
        RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey, serializedValue, interceptCallback, remainingWaitMs);
        if (result.batchIsFull || result.newBatchCreated) {
            log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), partition);
            this.sender.wakeup();
        }
        return result.future;
        // handling exceptions and record the errors;
        // for API exceptions return them in the future,
        // for other exceptions throw directly
    } catch (Exception e) {
        // ....
    }
}
```

关键过程如下：

1. 调用ProducerInterceptors.onSend()方法，通过ProducerInterceptor对消息进行拦截或修改。
2. 调用waitOnMetadata()方法获取Kafka集群的信息，底层会唤醒Send线程更新Metadata中保存的Kafka集群元数据。
3. 调用Serializer.serialize()方法序列化消息的key和value。
4. 调用partition()为消息选择合适的分区。
5. 调用RecordAccumulator.append()方法，将消息追加到RecordAccumulator中。
6. 唤醒Sender线程，由Sender线程将RecordAccumulator中缓存的消息发送出去。


### ProducerInterceptors＆ProducerInterceptor

面向切面编程的方法，重写 onSend onAcknowledgement onSendError 方法，添加逻辑。

ProducerInterceptors 则是类似组合模式。

### Metadata

Metadata 中有个 Cluster 类，Cluster 中有 Node TopicPartition PartitionInfo 三个类，共同组成了元数据信息。

Metadata 中的核心元素包括，topics version metadataExpireMs 等。

注意他的方法，例如 awaitUpdate 都是同步方法，里面还会有 wait 等方法。

### RecordAccumulator

主线程 的 KafkaProducer.send() 方法，实际上是放到了 RecordAccumulator 中。
一般的设计都是这样，Hdfs 中写数据也是先放到一个 chunk，然后累计为 packet 再一起发送。

主要的类包括 ArrayDeque＜RecordBatch＞，RecordBatch 中有 MemoryRecords 存放消息。

#### MemoryRecords

主要的字段：

```java
// 存放的数据
private ByteBuffer buffer;
// 最多存放的数据量
private final int writeLimit;
// 压缩数据
private final Compressor compressor;
// 此时是只读模式还是可写模式，发送数据前会设置为只读模式
private boolean writable;
```

MemoryRecords 只有私有构造方法，只能通过 emptyRecords() 方法得到对象。主要的方法：

```java
append()方法：先判断 MemoryRecords 是否为可写模式，然后调用 Compressor. put*() 方法，将消息数据写入 ByteBuffer 中。

hasRoomFor()方法：根据 Compressor 估算的已写字节数，估计 MemoryRecords 剩余空间是否足够写入指定的数据。
注意，这里仅仅是估算，所以不一定准确，通过hasRoomFor()方法判断之后写入数据，也可能就会导致底层ByteBuffer出现扩容的情况。

close()方法：出现 ByteBuffer 扩容的情况时，MemoryRecords.buffer 字段与 ByteBufferOutputStream.buffer 字段
所指向的不再是同一个 ByteBuffer 对象，在 close() 方法中，会将MemoryRecords.buffer字段指向扩容后的ByteBuffer对象，
同时，将 writable 设置为 false（即只读模式）。

sizeInBytes()方法：对于可写的 MemoryRecords，返回的是 ByteBufferOutputStream.buffer 字段的大小；对于只读 MemoryRecords，返回的是 MemoryRecords.buffer 的大小。
```

#### RecordBatch

听名字就知道这是一批 record， 每个 RecordBatch 中都有一个 MemoryRecords 字段。以及很多控制信息和统计信息。
主要的属性是：

```java

recordCount 	：记录了保存的Record的个数。
maxRecordSize	：最大Record的字节数。
attempts		：尝试发送当前RecordBatch的次数。
lastAttemptMs	：最后一次尝试发送的时间戳。
records			：指向用来存储数据的MemoryRecords对象。
topicPartition	：当前RecordBatch中缓存的消息都会发送给此TopicPartition。
produceFuture	：ProduceRequestResult类型，标识RecordBatch状态的Future对象。
lastAppendTime	：最后一次向RecordBatch追加消息的时间戳。
thunks			：Thunk对象的集合，可能比较重要，慢慢讨论
offsetCounter	：用来记录某消息在RecordBatch中的偏移量。
retry			：是否正在重试。如果RecordBatch中的数据发送失败，则会重新尝试发送。
```

核心方法是 tryAppend 和 RecordBatch.done

```java
public FutureRecordMetadata tryAppend(long timestamp, byte[] key, byte[] value, Callback callback, long now) {
    if (!this.records.hasRoomFor(key, value)) {
        return null;
    } else {
        long checksum = this.records.append(offsetCounter++, timestamp, key, value);
        this.maxRecordSize = Math.max(this.maxRecordSize, Record.recordSize(key, value));
        this.lastAppendTime = now;
        FutureRecordMetadata future = new FutureRecordMetadata(this.produceFuture, this.recordCount,
                                                               timestamp, checksum,
                                                               key == null ? -1 : key.length,
                                                               value == null ? -1 : value.length);
        if (callback != null)
            thunks.add(new Thunk(callback, future));
        this.recordCount++;
        return future;
    }
}

public void done(long baseOffset, long timestamp, RuntimeException exception) {
    log.trace("Produced messages to topic-partition {} with base offset offset {} and error: {}.",
              topicPartition,
              baseOffset,
              exception);
    // execute callbacks
    for (int i = 0; i < this.thunks.size(); i++) {
        try {
            Thunk thunk = this.thunks.get(i);
            if (exception == null) {
                // If the timestamp returned by server is NoTimestamp, that means CreateTime is used. Otherwise LogAppendTime is used.
                RecordMetadata metadata = new RecordMetadata(this.topicPartition,  baseOffset, thunk.future.relativeOffset(),
                                                             timestamp == Record.NO_TIMESTAMP ? thunk.future.timestamp() : timestamp,
                                                             thunk.future.checksum(),
                                                             thunk.future.serializedKeySize(),
                                                             thunk.future.serializedValueSize());
                thunk.callback.onCompletion(metadata, null);
            } else {
                thunk.callback.onCompletion(null, exception);
            }
        } catch (Exception e) {
            log.error("Error executing user-provided callback on message for topic-partition {}:", topicPartition, e);
        }
    }
    this.produceFuture.done(topicPartition, baseOffset, exception);
}
```

#### BufferPool

ByteBuffer 频繁创建会耗时，所以会有内存管理模块 BufferPool。
BufferPool 值管理固定大小的内存，一般会条件 MemoryRecords 的 buffer 大小可以缓存多条消息。
如果消息太大了，MemoryRecords 容量不够，会额外分配 ByteBuffer，使用完以后也是直接交给 GC 回收，
BufferPool 中的关键字段：

```java

free：是一个ArrayDeque＜ByteBuffer＞队列，其中缓存了指定大小的ByteBuffer对象。
ReentrantLock：因为有多线程并发分配和回收 ByteBuffer，所以使用锁控制并发，保证线程安全。
waiters：记录因申请不到足够空间而阻塞的线程，此队列中实际记录的是阻塞线程对应的 Condition 对象。
totalMemory：记录了整个 Pool 的大小。
availableMemory：记录了可用的空间大小，这个空间是 totalMemory 减去 free 列表中全部 ByteBuffer 的大小。
 
```

关键方法就是 BufferPool.allocate，里面有类似生产者消费者模式，这里不展示，如果想了解 java NIO 后续可以花时间分析。

#### RecordAccumulator 

关键的字段：

```java
batches：TopicPartition与RecordBatch集合的映射关系，类型是CopyOnWriteMap，是线程安全的集合，但其中的Deque是ArrayDeque类型，是非线程安全的集合。在后面的介绍中可以看到，追加新消息或发送RecordBatch的时候，需要加锁同步。每个Deque中都保存了发往对应TopicPartition的RecordBatch集合。
batchSize：指定每个 RecordBatch 底层 ByteBuffer 的大小。
Compression：压缩类型，参考 MemoryRecords 。
incomplete：未发送完成的 RecordBatch 集合，底层通过 Set＜RecordBatch＞ 集合实现。
free：BufferPool对象，参考 BufferPool 。
drainIndex：使用drain方法批量导出 RecordBatch 时，为了防止饥饿，使用 drainIndex 记录上次发送停止时的位置，下次继续从此位置开始发送。
```

关键方法 RecordsAccumulator.append ready drain

首先是 append 方法，用来给 deque 追加数据
```java
public RecordAppendResult append(TopicPartition tp,
                                 long timestamp,
                                 byte[] key,
                                 byte[] value,
                                 Callback callback,
                                 long maxTimeToBlock) throws InterruptedException {
    // We keep track of the number of appending thread to make sure we do not miss batches in
    // abortIncompleteBatches().
    // 记录 append 的 thread 数量，保证在 abortIncompleteBatches 方法中不会丢失 batches
    appendsInProgress.incrementAndGet();
    try {
        // 分拣思路，有就 get ，没有就 set，注意内部有同步控制。
        Deque<RecordBatch> dq = getOrCreateDeque(tp);
        
        // 加锁，然后向内部 tryAppend，tryAppend 实际上就是向 dq 队列最后一个 RecordBatch 添加。
        // 如果 dq 是空的，会添加失败
        synchronized (dq) {
            if (closed)
                throw new IllegalStateException("Cannot send after the producer is closed.");
            RecordAppendResult appendResult = tryAppend(timestamp, key, value, callback, dq);
            if (appendResult != null)
                return appendResult;
        }

        // we don't have an in-progress record batch try to allocate a new batch
        int size = Math.max(this.batchSize, Records.LOG_OVERHEAD + Record.recordSize(key, value));
        log.trace("Allocating a new {} byte message buffer for topic {} partition {}", size, tp.topic(), tp.partition());
        
        // 分配一段空间
        ByteBuffer buffer = free.allocate(size, maxTimeToBlock);
        synchronized (dq) { // 再次尝试 tryAppend 这个类似 单例模式的双重检测锁。
            // Need to check if producer is closed again after grabbing the dequeue lock.
            if (closed)
                throw new IllegalStateException("Cannot send after the producer is closed.");

            RecordAppendResult appendResult = tryAppend(timestamp, key, value, callback, dq);
            if (appendResult != null) {
                // Somebody else found us a batch, return the one we waited for! Hopefully this doesn't happen often...
                // 成功了就释放内存，返回
                free.deallocate(buffer);
                return appendResult;
            }
            
            // 没有并发操作，就老老实实 新建、append、添加、返回
            MemoryRecords records = MemoryRecords.emptyRecords(buffer, compression, this.batchSize);
            RecordBatch batch = new RecordBatch(tp, records, time.milliseconds());
            FutureRecordMetadata future = Utils.notNull(batch.tryAppend(timestamp, key, value, callback, time.milliseconds()));

            dq.addLast(batch);
            incomplete.add(batch);
            
            // dq.size() > 1 || batch.records.isFull() 的意思是 dq 有数据或者 batch满了，这个是用作后续的唤醒写程序
            return new RecordAppendResult(future, dq.size() > 1 || batch.records.isFull(), true);
        }
    } finally {
        appendsInProgress.decrementAndGet();
    }
}
```

append 代码逻辑很清晰，不过这里为什么要用双重检测锁而不是一次同步方法？
BufferPool 申请新的 ByteBuffer 的时候可能遇到阻塞，假如 线程1 消息比较大导致阻塞，线程2 消息小不会阻塞，
但是由于 线程1 获得了锁，线程2 也要阻塞。

这里体现了 减少锁的持有时间 的优化手段。

然后会调用 ready 获得准备好的 节点，这里的准备好指的是发送的数据的分区锁存储的节点：

```java
public ReadyCheckResult ready(Cluster cluster, long nowMs) {
    
    // 准备好的节点
    Set<Node> readyNodes = new HashSet<>();
    // 下一次调用 ready 的时间间隔
    long nextReadyCheckDelayMs = Long.MAX_VALUE;
    // 是否有 unknown 的 leader，如果有会触发 Metadata 更新。
    boolean unknownLeadersExist = false;

	// 饥饿？free 中有线程阻塞等待释放空间
    boolean exhausted = this.free.queued() > 0;
    
    // 遍历
    for (Map.Entry<TopicPartition, Deque<RecordBatch>> entry : this.batches.entrySet()) {
        TopicPartition part = entry.getKey();
        Deque<RecordBatch> deque = entry.getValue();

        Node leader = cluster.leaderFor(part);
        if (leader == null) {
            unknownLeadersExist = true;
        } else if (!readyNodes.contains(leader) && !muted.contains(part)) {
            synchronized (deque) {
                RecordBatch batch = deque.peekFirst();
                if (batch != null) {
                	// 5 个条件，满足一个就会更新
                    boolean backingOff = batch.attempts > 0 && batch.lastAttemptMs + retryBackoffMs > nowMs;
                    long waitedTimeMs = nowMs - batch.lastAttemptMs;
                    long timeToWaitMs = backingOff ? retryBackoffMs : lingerMs;
                    long timeLeftMs = Math.max(timeToWaitMs - waitedTimeMs, 0);
                    boolean full = deque.size() > 1 || batch.records.isFull();
                    boolean expired = waitedTimeMs >= timeToWaitMs;
                    boolean sendable = full || expired || exhausted || closed || flushInProgress();
                    if (sendable && !backingOff) {
                        readyNodes.add(leader);
                    } else {
                        // Note that this results in a conservative estimate since an un-sendable partition may have
                        // a leader that will later be found to have sendable data. However, this is good enough
                        // since we'll just wake up and then sleep again for the remaining time.
                        nextReadyCheckDelayMs = Math.min(timeLeftMs, nextReadyCheckDelayMs);
                    }
                }
            }
        }
    }

    return new ReadyCheckResult(readyNodes, nextReadyCheckDelayMs, unknownLeadersExist);
}
```

ready 代码也很清晰，注意就是计算 readyNodes。

计算好了 readyNodes ，drain 方法会获得对应的 NodeId 和 list[RecordBatch] 映射，为什么要将 TopicPartition 转换为 NodeId？
因为 producer 发送的时候只关心 TopicPartition， 但是真正发送的时候需要判断发送到哪个 Nodes。

```java
public Map<Integer, List<RecordBatch>> drain(Cluster cluster,
                                             Set<Node> nodes,
                                             int maxSize,
                                             long now) {
    if (nodes.isEmpty())
        return Collections.emptyMap();
	
	// 转换后的结果，也就是最终返回的数据
    Map<Integer, List<RecordBatch>> batches = new HashMap<>();
    for (Node node : nodes) {
        int size = 0;
        List<PartitionInfo> parts = cluster.partitionsForNode(node.id());
        List<RecordBatch> ready = new ArrayList<>();
        /* to make starvation less likely this loop doesn't start at 0 */
        int start = drainIndex = drainIndex % parts.size();
        do {
            PartitionInfo part = parts.get(drainIndex);
            TopicPartition tp = new TopicPartition(part.topic(), part.partition());
            // Only proceed if the partition has no in-flight batches.
            if (!muted.contains(tp)) {
                Deque<RecordBatch> deque = getDeque(new TopicPartition(part.topic(), part.partition()));
                if (deque != null) {
                    synchronized (deque) {
                        RecordBatch first = deque.peekFirst();
                        if (first != null) {
                            boolean backoff = first.attempts > 0 && first.lastAttemptMs + retryBackoffMs > now;
                            // Only drain the batch if it is not during backoff period.
                            if (!backoff) {
                            	// 如果发送的长度太长，而且 ready 已经有数据
                                if (size + first.records.sizeInBytes() > maxSize && !ready.isEmpty()) {
                                    // there is a rare case that a single batch size is larger than the request size due
                                    // to compression; in this case we will still eventually send this batch in a single
                                    // request
                                    break;
                                } else {
                                	// 发送长度没有达到极限，或者达到极限但是还没有数据，也就是一条数据长度就超过了极限，仍然发送一次
                                    RecordBatch batch = deque.pollFirst();
                                    batch.records.close();
                                    size += batch.records.sizeInBytes();
                                    ready.add(batch);
                                    batch.drainedMs = now;
                                }
                            }
                        }
                    }
                }
            }
            this.drainIndex = (this.drainIndex + 1) % parts.size();
        } while (start != drainIndex);
        batches.put(node.id(), ready);
    }
    return batches;
}
```

#### debug 

我们大概写个 demo debug 一下，增加我们的理解，写之前我们先大概搞清楚几个类的关系，如下的树状图：

```java
KafkaProducer -> RecordAccumulator 
	RecordAccumulator -> BufferPool free
		BufferPool -> Deque<ByteBuffer> free
	RecordAccumulator -> ConcurrentMap[TopicPartition, Deque[RecordBatch]] batches
		RecordBatch -> MemoryRecords
			MemoryRecords -> Compressor
				Compressor -> ByteBuffer
```

发送数据的过程，大概是 RecordAccumulator 按照分区发送数据到 Deque 中，
Deque 中的某一个 RecordBatch 一旦 full 了，或者 Deque.size()>1 了，就会触发 sender 线程。

获得元数据的调用栈：

```java
at org.apache.kafka.clients.Metadata.awaitUpdate(Metadata.java:129)
	  - locked <0x572> (a org.apache.kafka.clients.Metadata)
	  at org.apache.kafka.clients.producer.KafkaProducer.waitOnMetadata(KafkaProducer.java:528)
	  at org.apache.kafka.clients.producer.KafkaProducer.doSend(KafkaProducer.java:441)
	  at org.apache.kafka.clients.producer.KafkaProducer.send(KafkaProducer.java:430)
	  at org.apache.kafka.clients.producer.KafkaProducer.send(KafkaProducer.java:353)
	
```

分配内存调用栈：

```java
"main@1" prio=5 tid=0x1 nid=NA runnable
  java.lang.Thread.State: RUNNABLE
	  at org.apache.kafka.clients.producer.internals.BufferPool.allocate(BufferPool.java:99)
	  at org.apache.kafka.clients.producer.internals.RecordAccumulator.append(RecordAccumulator.java:181)
	  at org.apache.kafka.clients.producer.KafkaProducer.doSend(KafkaProducer.java:467)
	  at org.apache.kafka.clients.producer.KafkaProducer.send(KafkaProducer.java:430)
	  at org.apache.kafka.clients.producer.KafkaProducer.send(KafkaProducer.java:353)
```

发数据的调用栈：

```java
"main@1" prio=5 tid=0x1 nid=NA runnable
  java.lang.Thread.State: RUNNABLE
	 blocks kafka-producer-network-thread | DemoProducer@1371
	  at org.apache.kafka.common.record.MemoryRecords.append(MemoryRecords.java:97)
	  at org.apache.kafka.clients.producer.internals.RecordBatch.tryAppend(RecordBatch.java:70)
	  at org.apache.kafka.clients.producer.internals.RecordAccumulator.append(RecordAccumulator.java:195)
	  - locked <0x6be> (a java.util.ArrayDeque)
	  at org.apache.kafka.clients.producer.KafkaProducer.doSend(KafkaProducer.java:467)
	  at org.apache.kafka.clients.producer.KafkaProducer.send(KafkaProducer.java:430)
```


ready 调用的线程:

```java
"main@1" prio=5 tid=0x1 nid=NA waiting
  java.lang.Thread.State: WAITING
	  at sun.misc.Unsafe.park(Unsafe.java:-1)
	  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	  at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:836)
	  at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireSharedInterruptibly(AbstractQueuedSynchronizer.java:997)
	  at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1304)
	  at java.util.concurrent.CountDownLatch.await(CountDownLatch.java:231)
	  at org.apache.kafka.clients.producer.internals.ProduceRequestResult.await(ProduceRequestResult.java:57)
	  at org.apache.kafka.clients.producer.internals.FutureRecordMetadata.get(FutureRecordMetadata.java:51)
	  at org.apache.kafka.clients.producer.internals.FutureRecordMetadata.get(FutureRecordMetadata.java:25)
	  at org.apache.kafka.clients.producer.ProducerDemo.testSendMessage(ProducerDemo.java:45)
	  

"kafka-producer-network-thread | DemoProducer@1370" daemon prio=5 tid=0xd nid=NA runnable
  java.lang.Thread.State: RUNNABLE
	  at org.apache.kafka.clients.producer.internals.RecordAccumulator.ready(RecordAccumulator.java:306)
	  at org.apache.kafka.clients.producer.internals.Sender.run(Sender.java:175)
	  at org.apache.kafka.clients.producer.internals.Sender.run(Sender.java:134)
	  at java.lang.Thread.run(Thread.java:745)
```

drain 调用栈

```
"main@1" prio=5 tid=0x1 nid=NA waiting
  java.lang.Thread.State: WAITING
	  at sun.misc.Unsafe.park(Unsafe.java:-1)
	  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	  at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:836)
	  at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireSharedInterruptibly(AbstractQueuedSynchronizer.java:997)
	  at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1304)
	  at java.util.concurrent.CountDownLatch.await(CountDownLatch.java:231)
	  at org.apache.kafka.clients.producer.internals.ProduceRequestResult.await(ProduceRequestResult.java:57)
	  at org.apache.kafka.clients.producer.internals.FutureRecordMetadata.get(FutureRecordMetadata.java:51)
	  at org.apache.kafka.clients.producer.internals.FutureRecordMetadata.get(FutureRecordMetadata.java:25)
	  at org.apache.kafka.clients.producer.ProducerDemo.testSendMessage(ProducerDemo.java:45)

"kafka-producer-network-thread | DemoProducer@1370" daemon prio=5 tid=0xd nid=NA runnable
  java.lang.Thread.State: RUNNABLE
	  at org.apache.kafka.clients.producer.internals.RecordAccumulator.drain(RecordAccumulator.java:370)
	  at org.apache.kafka.clients.producer.internals.Sender.run(Sender.java:193)
	  at org.apache.kafka.clients.producer.internals.Sender.run(Sender.java:134)
```



其实有很多细节值得仔细分析和学习，后续慢慢分析学习吧。

### Sender

上面我们最后发现会调用 Sender 线程，我们分析一下。
Sender 根据 RecordAccumulator 的缓存情况调用 ready 方法 得到 NodeList，给 每个 Node 生成一个 请求发送出去。

关键属性就是：RecordAccumulator MetaData KafkaClient。

Sender 关键方法是 Runnable 的 run，run 方法就是调用了 ready drain createProduceRequests sendRequest

createProduceRequests 就是讲各种数据封装，每个 Node 无论有多少个 RecordBatch，都只有一个 Request。
Request 的内容就是将 api_key、version、client_id、record_set 等合在一起。代码这里不展示了。


#### KSelector

发送实际上是 NetworkClient 完成的，内部有一个 Selector，这个不是 java.nio 的 Selector，而是 kafka common 包中的。
我们暂且称之为 KSelector。它的注释是一个 non-blocking multi-connection network I/O，学习这个之前可以先了解一下 java.nio 的知识。

主要的属性是 nioSelector 和 channels 。
方法比较多

connect 

```java
public void connect(String id, InetSocketAddress address, int sendBufferSize, int receiveBufferSize) throws IOException {
    if (this.channels.containsKey(id))
        throw new IllegalStateException("There is already a connection for id " + id);

	// SocketChannel 的 使用
    SocketChannel socketChannel = SocketChannel.open();
    socketChannel.configureBlocking(false);
    Socket socket = socketChannel.socket();
    socket.setKeepAlive(true);
    if (sendBufferSize != Selectable.USE_DEFAULT_BUFFER_SIZE)
        socket.setSendBufferSize(sendBufferSize);
    if (receiveBufferSize != Selectable.USE_DEFAULT_BUFFER_SIZE)
        socket.setReceiveBufferSize(receiveBufferSize);
    socket.setTcpNoDelay(true);
    boolean connected;
    try {
        connected = socketChannel.connect(address);
    } catch (UnresolvedAddressException e) {
        socketChannel.close();
        throw new IOException("Can't resolve address: " + address, e);
    } catch (IOException e) {
        socketChannel.close();
        throw e;
    }
    
    // channel 注册。
    SelectionKey key = socketChannel.register(nioSelector, SelectionKey.OP_CONNECT);
    KafkaChannel channel = channelBuilder.buildChannel(id, key, maxReceiveSize);
    key.attach(channel);
    this.channels.put(id, channel);

    if (connected) {
        // OP_CONNECT won't trigger for immediately connected channels
        log.debug("Immediately connected to node {}", channel.id());
        immediatelyConnectedKeys.add(key);
        key.interestOps(0);
    }
}
```

比较关键的就是 `socketChannel.register(nioSelector, SelectionKey.OP_CONNECT)` 注册方法。

然后是 send 方法：

```java
    public void send(Send send) {
        KafkaChannel channel = channelOrFail(send.destination());
        try {
            channel.setSend(send);
        } catch (CancelledKeyException e) {
            this.failedSends.add(send.destination());
            close(channel);
        }
    }
```

可以看出 send 方法并没有发生网络 IO，而是将 send 对象放进 channel 中，等待 poll 方法才会发送出去。
每个 poll  只会发送一个请求。

```java
public void poll(long timeout) throws IOException {
    if (timeout < 0)
        throw new IllegalArgumentException("timeout should be >= 0");

    clear();

    if (hasStagedReceives() || !immediatelyConnectedKeys.isEmpty())
        timeout = 0;

    /* check ready keys */
    long startSelect = time.nanoseconds();
    int readyKeys = select(timeout);
    long endSelect = time.nanoseconds();
    currentTimeNanos = endSelect;
    this.sensors.selectTime.record(endSelect - startSelect, time.milliseconds());

    if (readyKeys > 0 || !immediatelyConnectedKeys.isEmpty()) {
        pollSelectionKeys(this.nioSelector.selectedKeys(), false);
        pollSelectionKeys(immediatelyConnectedKeys, true);
    }

    addToCompletedReceives();

    long endIo = time.nanoseconds();
    this.sensors.ioTime.record(endIo - endSelect, time.milliseconds());
    maybeCloseOldestConnection();
}
```

poll 方法调用 nioSelector.select() 然后调用 nioSelector.selectedKeys() 得到 selectedKeys，
然后调用 pollSelectionKeys，KSelector.pollSelectionKeys 是核心方法。

方法比较长这里不展示，主要是调用了 KafkaChannel 的 write 方法。

```
private boolean send(Send send) throws IOException {
    send.writeTo(transportLayer);
    if (send.completed())
        transportLayer.removeInterestOps(SelectionKey.OP_WRITE);

    return send.completed();
}
```

#### InFlightRequests

InFlightRequests 通过一个 Map＜String, Deque＜ClientRequest＞＞ 缓存了已经发送出去但是没有收到回复的 ClientRequest。

#### MetadataUpdater

辅助 NetworkClient 更新 Metadata 的类。

#### NetworkClient

NetworkClient.ready()方法用来检查Node是否准备好接收数据。


## 写数据流程

首先新建一个 ProducerRecord，ProducerRecord 的字段有 String topic, Integer partition, Long timestamp, K key, V value

```java

-- 第一步
producer.send(new ProducerRecord<>(topic,
                        messageNo,
                        messageStr)).get();

```

然后 调用 KafkaProducer 的 doSend 方法。

```java
private Future<RecordMetadata> doSend(ProducerRecord<K, V> record, Callback callback) 
```

会进行一系列处理，首先是获取 Metadata，然后序列化，计算这条消息发送到哪个分区。
调用： `accumulator.append(tp, timestamp, serializedKey, serializedValue, interceptCallback, remainingWaitMs)`

```
public RecordAppendResult append(TopicPartition tp,
                                 long timestamp,
                                 byte[] key,
                                 byte[] value,
                                 Callback callback,
                                 long maxTimeToBlock) throws InterruptedException
```

其中 TopicPartition 包含了 topic、和 partition 。`TopicPartition(String topic, int partition)`
这个方法会将数据 保存到 `ConcurrentMap<TopicPartition, Deque<RecordBatch>> batches` 。
具体就是先根据 TopicPartition 找到对应的 Deque<RecordBatch>，调用 peekLast() 得到最后面的 RecordBatch，
然后调用它的 `tryAppend(timestamp, key, value, callback, time.milliseconds())` 方法。

这里有几个注意的地方，第一个是 RecordBatch 的新建过程 `MemoryRecords.emptyRecords(buffer, compression, this.batchSize)`，
第二个是 这里需要加锁防止并发错误，类似 双重检测锁。

`private MemoryRecords(ByteBuffer buffer, CompressionType type, boolean writable, int writeLimit)`
可以看出 MemoryRecords 最重要的就是 ByteBuffer 字段，
`new RecordBatch(tp, records, time.milliseconds())` 可以看出 RecordBatch 重要的就是 topicPartition 和 MemoryRecords 字段。

最终 MemoryRecords 的 append 方法，`this.records.append(offsetCounter++, timestamp, key, value)`，
这里有个细节，offsetCounter++，也就是每个 records 内部的 offset 都是相对于这个 batch 的。

```java
public long append(long offset, long timestamp, byte[] key, byte[] value) {
    if (!writable)
        throw new IllegalStateException("Memory records is not writable");

    int size = Record.recordSize(key, value);
    compressor.putLong(offset);
    compressor.putInt(size);
    long crc = compressor.putRecord(timestamp, key, value);
    compressor.recordWritten(size + Records.LOG_OVERHEAD);
    return crc;
}
```

最终写的数据是：
```java
public static void write(Compressor compressor, long crc, byte attributes, long timestamp, byte[] key, byte[] value, int valueOffset, int valueSize) {
    // write crc
    compressor.putInt((int) (crc & 0xffffffffL));
    // write magic value
    compressor.putByte(CURRENT_MAGIC_VALUE);
    // write attributes
    compressor.putByte(attributes);
    // write timestamp
    compressor.putLong(timestamp);
    // write the key
    if (key == null) {
        compressor.putInt(-1);
    } else {
        compressor.putInt(key.length);
        compressor.put(key, 0, key.length);
    }
    // write the value
    if (value == null) {
        compressor.putInt(-1);
    } else {
        int size = valueSize >= 0 ? valueSize : (value.length - valueOffset);
        compressor.putInt(size);
        compressor.put(value, valueOffset, size);
    }
}
```
最后服务端收到数据以后只要反过来解析即可。

我们的数据都在 RecordBatch 中， Sender 的 createProduceRequests 方法能解析出来：

```
/**
 * Transfer the record batches into a list of produce requests on a per-node basis
 */
private List<ClientRequest> createProduceRequests(Map<Integer, List<RecordBatch>> collated, long now) {
    List<ClientRequest> requests = new ArrayList<ClientRequest>(collated.size());
    for (Map.Entry<Integer, List<RecordBatch>> entry : collated.entrySet())
        requests.add(produceRequest(now, entry.getKey(), acks, requestTimeout, entry.getValue()));
    return requests;
}

/**
 * Create a produce request from the given record batches
 */
private ClientRequest produceRequest(long now, int destination, short acks, int timeout, List<RecordBatch> batches) {
    Map<TopicPartition, ByteBuffer> produceRecordsByPartition = new HashMap<TopicPartition, ByteBuffer>(batches.size());
    final Map<TopicPartition, RecordBatch> recordsByPartition = new HashMap<TopicPartition, RecordBatch>(batches.size());
    for (RecordBatch batch : batches) {
        TopicPartition tp = batch.topicPartition;
        produceRecordsByPartition.put(tp, batch.records.buffer());
        recordsByPartition.put(tp, batch);
    }
    ProduceRequest request = new ProduceRequest(acks, timeout, produceRecordsByPartition);
    RequestSend send = new RequestSend(Integer.toString(destination),
                                       this.client.nextRequestHeader(ApiKeys.PRODUCE),
                                       request.toStruct());
    RequestCompletionHandler callback = new RequestCompletionHandler() {
        public void onComplete(ClientResponse response) {
            handleProduceResponse(response, recordsByPartition, time.milliseconds());
        }
    };

    return new ClientRequest(now, acks != 0, send, callback);
}
```


