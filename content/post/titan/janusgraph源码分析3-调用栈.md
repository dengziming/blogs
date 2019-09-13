---
date: 2018-03-22
title: "janusgraph源码分析3-调用栈"
author: "邓子明"
tags:
    - janusgraph
    - 源码
categories:
    - 源码分析
comment: true
---


我们可以在比较关键的地方大断点，然后分析整个调用栈，进行进一步分析。哪里是关键点是需要一定经验判断的。

例如我们基于 hadoop spark 等框架的时候，我们写的代码就是关键的，打断点可以看到合适调用，怎么被调用。
我们关心怎么写数据，可以在和底层数据交互的地方打断点。总之我们关心谁就在哪里打断点。

记住：打断点的地方基本上是最终的调用点。

## 整体调试找关键

首先是存储类，我们使用本地文件存储，存储使用类是：`com.sleepycat.je.Database` 这个类具体功能是啥可以具体研究。我们发现它有 get delete put 等方法，我们可以打上断点。然后查看调用栈。

得到 普通 的调用信息：
```java
"main@1" prio=5 tid=0x1 nid=NA runnable
  java.lang.Thread.State: RUNNABLE
	  at com.sleepycat.je.Database.put(Database.java:1574)
	  at com.sleepycat.je.Database.put(Database.java:1627)
	  at org.janusgraph.diskstorage.berkeleyje.BerkeleyJEKeyValueStore.insert(BerkeleyJEKeyValueStore.java:195)
	  at org.janusgraph.diskstorage.berkeleyje.BerkeleyJEKeyValueStore.insert(BerkeleyJEKeyValueStore.java:184)
	  at org.janusgraph.diskstorage.keycolumnvalue.keyvalue.OrderedKeyValueStoreAdapter.mutate(OrderedKeyValueStoreAdapter.java:99)
	  at org.janusgraph.diskstorage.configuration.backend.KCVSConfiguration$2.call(KCVSConfiguration.java:154)
	  at org.janusgraph.diskstorage.configuration.backend.KCVSConfiguration$2.call(KCVSConfiguration.java:149)
	  at org.janusgraph.diskstorage.util.BackendOperation.execute(BackendOperation.java:147)
	  at org.janusgraph.diskstorage.util.BackendOperation$1.call(BackendOperation.java:161)
	  at org.janusgraph.diskstorage.util.BackendOperation.executeDirect(BackendOperation.java:68)
	  at org.janusgraph.diskstorage.util.BackendOperation.execute(BackendOperation.java:54)
	  at org.janusgraph.diskstorage.util.BackendOperation.execute(BackendOperation.java:158)
	  at org.janusgraph.diskstorage.configuration.backend.KCVSConfiguration.set(KCVSConfiguration.java:149)
	  at org.janusgraph.diskstorage.configuration.backend.KCVSConfiguration.set(KCVSConfiguration.java:126)
	  at org.janusgraph.diskstorage.configuration.ModifiableConfiguration.set(ModifiableConfiguration.java:40)
	  at org.janusgraph.diskstorage.configuration.ModifiableConfiguration.setAll(ModifiableConfiguration.java:47)
	  at org.janusgraph.graphdb.configuration.GraphDatabaseConfiguration.<init>(GraphDatabaseConfiguration.java:1266)
	  at org.janusgraph.core.JanusGraphFactory.open(JanusGraphFactory.java:160)
	  at org.janusgraph.core.JanusGraphFactory.open(JanusGraphFactory.java:131)
	  at org.janusgraph.core.JanusGraphFactory.open(JanusGraphFactory.java:78)
	  at org.janusgraph.test.dengziming.FirstTest.main(FirstTest.java:37)

```

从下往上可以看出，顺序：
```
new GraphDatabaseConfiguration
ModifiableConfiguration.setAll(getGlobalSubset(localBasicConfiguration.getAll())); 
KCVSConfiguration.set(key,value,null,false);
BackendOperation.execute(new BackendOperation.Transactional<Boolean>() {@Override public Boolean call}
然后调用 上面new 的 BackendOperation.Transactional 的 call 方法
然后是 store.mutate
status = db.put(tx, key.as(ENTRY_FACTORY), value.as(ENTRY_FACTORY));
put(txn, key, data, Put.OVERWRITE, null);
result = cursor.putInternal(key, data, putType, options);
最终调用的是 cursor.putNotify 插入数据。
```

这个 put 会多次调用，config 会设置 "startup-time" 等属性，都是通过这个put方法实现。

第二次用到这个方法是 创建 VertexLabel 的时候会分配 id， 这时候我们可以看一下更详细的调用栈：
```java
"JanusGraphID(0)(4)[0]@5358" prio=5 tid=0x24 nid=NA runnable
  java.lang.Thread.State: RUNNABLE
	  at com.sleepycat.je.dbi.CursorImpl.insertRecordInternal(CursorImpl.java:1364)
	  at com.sleepycat.je.dbi.CursorImpl.insertOrUpdateRecord(CursorImpl.java:1221)
	  at com.sleepycat.je.Cursor.putNoNotify(Cursor.java:2962)
	  at com.sleepycat.je.Cursor.putNotify(Cursor.java:2800)
	  at com.sleepycat.je.Cursor.putNoDups(Cursor.java:2647)
	  at com.sleepycat.je.Cursor.putInternal(Cursor.java:2478)
	  - locked <0x1536> (a com.sleepycat.je.Transaction)
	  at com.sleepycat.je.Cursor.putInternal(Cursor.java:830)
	  at com.sleepycat.je.Database.put(Database.java:1574)
	  at com.sleepycat.je.Database.put(Database.java:1627)
	  at org.janusgraph.diskstorage.berkeleyje.BerkeleyJEKeyValueStore.insert(BerkeleyJEKeyValueStore.java:195)
	  at org.janusgraph.diskstorage.berkeleyje.BerkeleyJEKeyValueStore.insert(BerkeleyJEKeyValueStore.java:184)
	  at org.janusgraph.diskstorage.keycolumnvalue.keyvalue.OrderedKeyValueStoreAdapter.mutate(OrderedKeyValueStoreAdapter.java:99)
	  at org.janusgraph.diskstorage.idmanagement.ConsistentKeyIDAuthority.lambda$getIDBlock$1(ConsistentKeyIDAuthority.java:261)
	  at org.janusgraph.diskstorage.idmanagement.ConsistentKeyIDAuthority$$Lambda$71.1795053717.call(Unknown Source:-1)
	  at org.janusgraph.diskstorage.util.BackendOperation.execute(BackendOperation.java:147)
	  at org.janusgraph.diskstorage.idmanagement.ConsistentKeyIDAuthority.getIDBlock(ConsistentKeyIDAuthority.java:260)
	  - locked <0x14f8> (a org.janusgraph.diskstorage.idmanagement.ConsistentKeyIDAuthority)
	  at org.janusgraph.graphdb.database.idassigner.StandardIDPool$IDBlockGetter.call(StandardIDPool.java:288)
	  at org.janusgraph.graphdb.database.idassigner.StandardIDPool$IDBlockGetter.call(StandardIDPool.java:255)
	  ...
	  at java.lang.Thread.run(Thread.java:745)
```

上面的调用栈没有显示这么多，实际上我们也没必要关心 `com.sleepycat.je.Database.put(Database.java:1627)` 之后的东西，
因为这些东西都是 数据库的写 API，而生产环境我们会使用 hbase和cassandra ，所以每次只要 debug 到 KeyColumnValueStore 的 相应方法即可，再 debug 就是数据库的方法。

到这里我们明白，增删改查都是 通过 KeyColumnValueStore 类完成。接下来我们直接在 BerkeleyJEKeyValueStore 的 增删改查方法 打断点就行。

###  management.commit();

management 是用来操作 schema 的类，我们可以猜测 schema 也是以系统属性的方式存在数据库中。通过打断点发现，前面的操作都没有触发 BerkeleyJEKeyValueStore 的insert ，直到 commit，
先取出调用栈：
```java
"main@1" prio=5 tid=0x1 nid=NA runnable
  java.lang.Thread.State: RUNNABLE
	  at org.janusgraph.diskstorage.berkeleyje.BerkeleyJEKeyValueStore.insert(BerkeleyJEKeyValueStore.java:195)
	  at org.janusgraph.diskstorage.berkeleyje.BerkeleyJEKeyValueStore.insert(BerkeleyJEKeyValueStore.java:184)
	  at org.janusgraph.diskstorage.berkeleyje.BerkeleyJEStoreManager.mutateMany(BerkeleyJEStoreManager.java:208)
	  at org.janusgraph.diskstorage.keycolumnvalue.keyvalue.OrderedKeyValueStoreManagerAdapter.mutateMany(OrderedKeyValueStoreManagerAdapter.java:125)
	  at org.janusgraph.diskstorage.keycolumnvalue.cache.CacheTransaction$1.call(CacheTransaction.java:94)
	  at org.janusgraph.diskstorage.keycolumnvalue.cache.CacheTransaction$1.call(CacheTransaction.java:91)
	  at org.janusgraph.diskstorage.util.BackendOperation.executeDirect(BackendOperation.java:68)
	  at org.janusgraph.diskstorage.util.BackendOperation.execute(BackendOperation.java:54)
	  at org.janusgraph.diskstorage.keycolumnvalue.cache.CacheTransaction.persist(CacheTransaction.java:91)
	  at org.janusgraph.diskstorage.keycolumnvalue.cache.CacheTransaction.flushInternal(CacheTransaction.java:139)
	  at org.janusgraph.diskstorage.keycolumnvalue.cache.CacheTransaction.commit(CacheTransaction.java:196)
	  at org.janusgraph.diskstorage.BackendTransaction.commitStorage(BackendTransaction.java:134)
	  at org.janusgraph.graphdb.database.StandardJanusGraph.commit(StandardJanusGraph.java:733)
	  at org.janusgraph.graphdb.transaction.StandardJanusGraphTx.commit(StandardJanusGraphTx.java:1372)
	  - locked <0x113a> (a org.janusgraph.graphdb.transaction.StandardJanusGraphTx)
	  at org.janusgraph.graphdb.database.management.ManagementSystem.commit(ManagementSystem.java:239)
	  - locked <0x102b> (a org.janusgraph.graphdb.database.management.ManagementSystem)
	  at org.janusgraph.example.GraphOfTheGodsFactory.load(GraphOfTheGodsFactory.java:111)

```
这里面好像还有锁，这个先不讨论。

主要的几个调用：

StandardJanusGraphTx.commit()

StandardJanusGraph.commit(addedRelations.getAll(), deletedRelations.values(), this); -- 这个 commit 的逻辑挺复杂，需要仔细查看。

BackendTransaction.commitStorage();

CacheTransaction.commit()

OrderedKeyValueStoreManagerAdapter.mutateMany

BerkeleyJEStoreManager.mutateMany(subMutations, tx);

BerkeleyJEKeyValueStore.insert();

然后接下来就是一个个分析这几个类每一个的属性和方法。

首先看一下类的继承结构

```
SchemaInspector	
	StandardJanusGraphTx (org.janusgraph.graphdb.transaction)
	SchemaManager (org.janusgraph.core.schema)
	    Transaction (org.janusgraph.core)
	        JanusGraphTransaction (org.janusgraph.core)
	            JanusGraphBlueprintsTransaction (org.janusgraph.graphdb.tinkerpop)
	                StandardJanusGraphTx (org.janusgraph.graphdb.transaction)
	        JanusGraph (org.janusgraph.core)
	            JanusGraphBlueprintsGraph (org.janusgraph.graphdb.tinkerpop)
	                StandardJanusGraph (org.janusgraph.graphdb.database)
	    JanusGraphManagement (org.janusgraph.core.schema)
	        ManagementSystem (org.janusgraph.graphdb.database.management)
```


SchemaInspector 接口定义了检查 schema 的一些方法，
例如：containsRelationType getRelationType containsPropertyKey getOrCreatePropertyKey getEdgeLabel getOrCreateVertexLabel 
这些方法有四类，分别是是 RelationType 相关的，PropertyKey 相关，EdgeLabel 相关，VertexLabel 相关。这四个代表啥大家应该都清楚了。

SchemaManager 接口 在 SchemaInspector 的基础上添加了 6 个方法 ：makePropertyKey makeEdgeLabel makeVertexLabel addProperties addProperties addConnection 。
其实前三个返回的是 Maker，后面三个返回的就是 Label。这六个方法左右主要是给 schema 添加更多信息，例如添加 properties。

Transaction 继承自 SchemaManager 和 Graph ，定义了 addVertex 和 query 等操作。很奇怪为什么只有 addVertex 没有 addEdge 和 addProperty 的操作。

JanusGraphManagement 继承自 SchemaManager 和 JanusGraphConfiguration ，定义了 buildEdgeIndex buildPropertyIndex commit 等操作
大部分都和 index 相关，例如构建查询更新。还有 getRelationTypes getVertexLabels 两个方法。

ManagementSystem 继承自 JanusGraphManagement ，通过代理 StandardJanusGraphTx ，实现了 getGraphIndex commit 等操作。

JanusGraphTransaction 继承自 Transaction ，定义了 addVertex getVertex commit rollback 等，和 Transaction 不同的是他的这些方法操作的都是 id，而后者操作的是 用户传入的 String

JanusGraphBlueprintsTransaction 继承自 JanusGraphTransaction ，目前看到的就是简单封装一下抽象方法，同时实现了 addVertex 方法。

StandardJanusGraphTx 继承自 JanusGraphBlueprintsTransaction ，实现了抽象的方法。

JanusGraph 继承自 Transaction， 定义了 buildTransaction openManagement close 等方法。

JanusGraphBlueprintsGraph 继承自 JanusGraph ，通过 ThreadLocal 实现线程隔离。
StandardJanusGraph 继承自 JanusGraphBlueprintsGraph 就是我们使用的 Graph 。

所以了解janus比较重要的是 StandardJanusGraphTx ，了解多线程的 JanusGraphBlueprintsGraph。

从继承结构大概可以看出所有的操作分为数据操作和 schema 操作，而分别由 JanusGraph 和 JanusGraphManagement 完成，实际上都是代理或者适配装饰了 StandardJanusGraphTx。StandardJanusGraphTx 内容很多。

### StandardJanusGraph

上面我们已经看出了实际上最重要的就是  StandardJanusGraphTx 的实现逻辑，我们就以他为入口，而不是 main 方法。它的构造方法里面需要用到 StandardJanusGraph ，我们先大概了解一下 。

#### 我们先看一下它的属性：
```
log
config
backend
idManager
idAssigner
times
indexSerializer
edgeSerializer
serializer
vertexExistenceQuery
queryCache
schemaCache
managementLogger
shutdownHook
isOpen
txCounter
openTransactions
name
typeCacheRetrieval
SCHEMA_FILTER
NO_SCHEMA_FILTER
NO_FILTER
```

GraphDatabaseConfiguration config 是图的配置，由于配置也是保存在数据库，所以也是需要访问数据库的。

Backend backend 是在 config.getBackend 中初始化的，Backend 的构造方法很复杂，主要创建出了 StoreManager indexes txLogManager 等管理存储很重要的属性。

idManager 和 idAssigner 都是和 id 相关的。 所属类为 IDManager ，VertexIDAssigner，有比较复杂的id分配算法。

IndexSerializer 和 EdgeSerializer 、Serializer 用于序列化，Serializer 在 config 中初始化，其他两个都是基于 Serializer 的封装。

vertexExistenceQuery:SliceQuery queryCache:RelationQueryCache schemaCache:SchemaCache 都是 cache 相关。

managementLogger 是 用来记录操作日志的。

typeCacheRetrieval ，看到 Retrieval 就知道是获取某些属性用的，他通过 `QueryUtil.getVertices(consistentTx, BaseKey.SchemaName, typeName)` 获得 JanusGraphVertex。


#### 然后再看方法

除了 getset 以外，主要是：

```
isOpen
isClosed
close
closeInternal
prepareCommit
commit
openManagement
newTransaction
buildTransaction
newThreadBoundTransaction
newTransaction
openBackendTransaction
closeTransaction
getVertexIDs
edgeQuery
edgeMultiQuery
assignID
assignID
acquireLock
acquireLock
getTTL
getTTL
```

和 transaction 有关的打开关闭提交等，查询边和顶点，分配id，获得锁。这里的 edgeQuery 并不是查询边，而是查询 edgestore 这个表格，这个表格存放了所有的数据。
细心分析发现，这些方法主要都是进行查询操作，得到查询结果 List<EntryList>，并没有进行数据增删改查的操作 API。

### ManagementSystem

StandardJanusGraph 用来操作数据，而 ManagementSystem 主要是管理 schema。

#### 属性：

```java
LOGGER
CURRENT_INSTANCE_SUFFIX
graph
sysLog
managementLogger
transactionalConfig
modifyConfig
userConfig
schemaCache
transaction
updatedTypes
evictGraphFromCache
updatedTypeTriggers
txStartTime
graphShutdownRequired
isOpen
configVerifier
```

graph 和 managementLogger 就是上面的 StandardJanusGraph 和 managementLogger。sysLog 也是和日志有关。

TransactionalConfiguration 是事务的配置，实际上他应该是记录了变化，能够判断是否有改变，从而进行 commit 和 rollback

SchemaCache 就是 StandardJanusGraph 的 SchemaCache。

transaction 是 StandardJanusGraphTx。

updatedTypes 应该也是记录更新

其他的暂时还不太懂。

#### 方法：

```
IndexBuilder
GraphCacheEvictionCompleteTrigger
EmptyIndexJobFuture
UpdateStatusTrigger
IndexJobStatus
IndexIdentifier
ManagementSystem

getOpenInstancesInternal
getOpenInstances
forceCloseInstance
ensureOpen
commit
rollback
isOpen
close
getWrappedTx
addSchemaEdge
getSchemaElement
buildEdgeIndex
buildEdgeIndex
buildPropertyIndex
buildPropertyIndex
buildRelationTypeIndex
composeRelationTypeIndexName
containsRelationIndex
getRelationIndex
getRelationIndexes
getGraphIndexDirect
containsGraphIndex
getGraphIndex
getGraphIndexes
awaitGraphIndexStatus
awaitRelationIndexStatus
checkIndexName
createMixedIndex
addIndexKey
createCompositeIndex
buildIndex
updateIndex
evictGraphFromCache
setUpdateTrigger
setStatus
setStatusVertex
setStatusEdges
getIndexJobStatus
changeName
updateConnectionEdgeConstraints
getSchemaVertex
updateSchemaVertex
getConsistency
setConsistency
getTTL
setTTL
setTypeModifier
containsRelationType
getRelationType
containsPropertyKey
getPropertyKey
containsEdgeLabel
getOrCreateEdgeLabel
getOrCreatePropertyKey
getEdgeLabel
makePropertyKey
makeEdgeLabel
getRelationTypes
containsVertexLabel
getVertexLabel
getOrCreateVertexLabel
makeVertexLabel
addProperties
addProperties
addConnection
getVertexLabels
get
set
```

强制关闭、操作事务、添加顶点边Label属性索引。

索引都是  buildRelationTypeIndex 方法，说明 RelationType(PropertyKey 和 EdgeLabel)才有索引，分别是 graphIndex 和 vertexIncdicentIndex ，VertexLabel 没有索引。
而 getVertexLabels 等带s的方法 是 调用 QueryUtil.getVertices ，说明得到所有的需要查询数据库。

很多方法都是直接调用 StandardJanusGraphTx 的 对应方法。但是 build Index 并没有使用到 StandardJanusGraphTx。说明 index 并不是马上就插入数据库？或者因为 Index 建完以后还要等待？？


### StandardJanusGraphTx

上面大致了解了  StandardJanusGraph 和 ManagementSystem ，StandardJanusGraphTx 内部才是最重要的，

#### 属性：

```java
log
EMPTY_DELETED_RELATIONS
UNINITIALIZED_LOCKS
LOCK_TIMEOUT
MIN_VERTEX_CACHE_SIZE
graph
config
idManager
idInspector
attributeHandler
txHandle
edgeSerializer
indexSerializer
vertexCache
addedRelations
deletedRelations
indexCache
newVertexIndexEntries
uniqueLocks
newTypeCache
temporaryIds
times
isOpen
existingVertexRetriever
externalVertexRetriever
internalVertexRetriever
edgeProcessor
edgeProcessorImpl
elementProcessor
elementProcessorImpl
vertexIDConversionFct
edgeIDConversionFct
propertyIDConversionFct
```

前面的属性都是在 graph 获得的

vertexCache = new GuavaVertexCache(effectiveVertexCacheSize,concurrencyLevel,config.getDirtyVertexSize()); 是缓存 vertex 的。

addedRelations = new ConcurrentBufferAddedRelations(); 是缓存 Relation 的。

deletedRelations 同上

indexCache 缓存 index ， 类似 vertexCache ，需要传入一个  retrival 

existingVertexRetriever externalVertexRetriever internalVertexRetriever 都是给 vertexCache 用来查 vertex 的。

edgeProcessor 是一个 QueryExecutor。用来查询的。

elementProcessor 一样是用来查询的。

#### 方法

```java
StandardJanusGraphTx
setBackendTransaction
verifyWriteAccess
verifyAccess
verifyOpen
getNextTx
getConfiguration
getGraph
getTxHandle
getEdgeSerializer
getIdInspector
isPartitionedVertex
getCanonicalVertex
getOtherPartitionVertex
getAllRepresentatives
containsVertex
isValidVertexId
getVertex
getVertices
getExistingVertex
getInternalVertex
addVertex
addVertex
addVertex
getInternalVertices
validDataType
verifyAttribute
removeRelation
isRemovedRelation
getLock
getLock
getUniquenessLock
checkPropertyConstraintForVertexOrCreatePropertyConstraint
checkPropertyConstraintForEdgeOrCreatePropertyConstraint
checkConnectionConstraintOrCreateConnectionConstraint
addEdge
connectRelation
addProperty
addProperty
getEdges
makeSchemaVertex
updateSchemaVertex
makePropertyKey
makeEdgeLabel
addSchemaEdge
addProperties
addProperties
addConnection
getSchemaVertex
containsRelationType
getRelationType
containsPropertyKey
containsEdgeLabel
getExistingRelationType
getPropertyKey
getOrCreatePropertyKey
getOrCreatePropertyKey
getEdgeLabel
getOrCreateEdgeLabel
makePropertyKey
makeEdgeLabel
getExistingVertexLabel
containsVertexLabel
getVertexLabel
getOrCreateVertexLabel
makeVertexLabel
query
multiQuery
multiQuery
executeMultiQuery
getConversionFunction
query
indexQuery
commit
rollback
releaseTransaction
isOpen
isClosed
hasModifications
```

schema 操作的 makeEdgeLabel makePropertyKey 等，数据操作的 getVertex addEdge 等，事务操作的 rollback 等。
好像没有 index ？因为 index 属于 schema， 相关的方法都是在 management 中完成的。
实际上，StandardJanusGraphTx 有 addEdge addProperties addVertex 等操作数据的方法，同时还有 makePropertyKey，EdgeLabel 等操作 schema 的方法。
原因是 makePropertyKey 等 schema 实际上也是以顶点的形式保存在 janus 中，所以 schema 操作本质还是数据操作，只不过这部分数据都会被读入内存。
所以 schema 操作都会触发 makeSchemaVertex 的方法，makeSchemaVertex 就是添加一个顶点，只不过是 schema 的订单。


### BackendTransaction

我们在看 StandardJanusGraphTx 代码的时候 ，发现 BackendTransaction 也很重要，看看他的继承体系
```
BaseTransaction
	LoggableTransaction (org.janusgraph.diskstorage)
	    CacheTransaction (org.janusgraph.diskstorage.keycolumnvalue.cache)
	    IndexTransaction (org.janusgraph.diskstorage.indexing)
	    BackendTransaction (org.janusgraph.diskstorage)
	BaseTransactionConfigurable (org.janusgraph.diskstorage)
	    Transaction in LuceneIndex (org.janusgraph.diskstorage.lucene)
	    DefaultTransaction (org.janusgraph.diskstorage.util)
	    StoreTransaction (org.janusgraph.diskstorage.keycolumnvalue)
	        AbstractStoreTransaction (org.janusgraph.diskstorage.common)
	            CQLTransaction (org.janusgraph.diskstorage.cql)
	            BerkeleyJETx (org.janusgraph.diskstorage.berkeleyje)
	            CassandraTransaction (org.janusgraph.diskstorage.cassandra)
	            HBaseTransaction (org.janusgraph.diskstorage.hbase)
	            NoOpStoreTransaction (org.janusgraph.diskstorage.common)
	            InMemoryTransaction in InMemoryStoreManager (org.janusgraph.diskstorage.keycolumnvalue.inmemory)
	        CacheTransaction (org.janusgraph.diskstorage.keycolumnvalue.cache)
	        ExpectedValueCheckingTransaction (org.janusgraph.diskstorage.locking.consistentkey)
	IndexTransaction (org.janusgraph.diskstorage.indexing)
```

BaseTransaction 只有 comimit 和 roolback 两个方法。LoggableTransaction 只有 LoggableTransaction ，BaseTransactionConfigurable 多了一个 getConfiguration 。

IndexTransaction BackendTransaction CacheTransaction 继承自 LoggableTransaction ， 前者是处理索引，后者可以处理其他的读写事务，最后的是内存中的事务处理。

IndexTransaction 中有一个 BaseTransaction 属性，用来实现真正的事务读写，实现一般是 IndexProvider 生成，主要是 ES、LUCENE、Solr 三种实现。
CacheTransaction 中有 StoreTransaction 属性，用来实现持久化。
BackendTransaction 中则有 CacheTransaction edgeStore indexStore txLogStore Map<String, IndexTransaction> indexTx; 等属性，显然这才是最重要的实现事务管控的类。

我们通过代码分析可以看出 BackendTransaction 的创建是在 StandardJanusGraph 完成，而使用主要是 StandardJanusGraphTx 。
StandardJanusGraph 的 newTransaction 创建 BackendTransaction 和 StandardJanusGraphTx ，并进行赋值。
StandardJanusGraph 什么时候会调用 newTransaction ？一个在 typeCacheRetrieval 中，另一个就是我们代码创建新的 transaction，还有一个是 在没有 transactional isolation 的存储系统上面， commit 的时候需要操作 schema



## 关键类分析

上面整体的调试已经找到了比较关键的大类，以及事务相关的类的关系，我们可以反过来再看一遍调用栈，就更清晰了。现在反过来从细节开始研究具体的类的功能。

### StoreManager

```java
StoreManager
	KeyValueStoreManager (org.janusgraph.diskstorage.keycolumnvalue.keyvalue)
	    OrderedKeyValueStoreManager (org.janusgraph.diskstorage.keycolumnvalue.keyvalue)
	        BerkeleyJEStoreManager (org.janusgraph.diskstorage.berkeleyje)
	AbstractStoreManager (org.janusgraph.diskstorage.common)
	    DistributedStoreManager (org.janusgraph.diskstorage.common)
	        CQLStoreManager (org.janusgraph.diskstorage.cql)
	            CachingCQLStoreManager (org.janusgraph.diskstorage.cql)
	        AbstractCassandraStoreManager (org.janusgraph.diskstorage.cassandra)
	            CassandraThriftStoreManager (org.janusgraph.diskstorage.cassandra.thrift)
	            CassandraEmbeddedStoreManager (org.janusgraph.diskstorage.cassandra.embedded)
	            AstyanaxStoreManager (org.janusgraph.diskstorage.cassandra.astyanax)
	        HBaseStoreManager (org.janusgraph.diskstorage.hbase)
	    LocalStoreManager (org.janusgraph.diskstorage.common)
	        BerkeleyJEStoreManager (org.janusgraph.diskstorage.berkeleyje)
	                    LocalStoreManagerSampleImplementation in LocalStoreManagerTest (org.janusgraph.diskstorage.common)
	            KeyColumnValueStoreManager (org.janusgraph.diskstorage.keycolumnvalue)
	    CQLStoreManager (org.janusgraph.diskstorage.cql)
	        CachingCQLStoreManager (org.janusgraph.diskstorage.cql)
	    OrderedKeyValueStoreManagerAdapter (org.janusgraph.diskstorage.keycolumnvalue.keyvalue)
	    InMemoryStoreManager (org.janusgraph.diskstorage.keycolumnvalue.inmemory)
	    KCVSManagerProxy (org.janusgraph.diskstorage.keycolumnvalue)
	        ExpectedValueCheckingStoreManager (org.janusgraph.diskstorage.locking.consistentkey)
	        TTLKCVSManager (org.janusgraph.diskstorage.keycolumnvalue.ttl)
	    MetricInstrumentedStoreManager (org.janusgraph.diskstorage.util)
	    AbstractCassandraStoreManager (org.janusgraph.diskstorage.cassandra)
	        CassandraThriftStoreManager (org.janusgraph.diskstorage.cassandra.thrift)
	        CassandraEmbeddedStoreManager (org.janusgraph.diskstorage.cassandra.embedded)
	        AstyanaxStoreManager (org.janusgraph.diskstorage.cassandra.astyanax)
	    HBaseStoreManager (org.janusgraph.diskstorage.hbase)
```

StoreManager 接口主要功能 beginTransaction 得到一个 StoreTransaction 和 close ，clean 等，还有得到 store 相关的信息。看着上面好像很多继承类，实际上是因为有重复继承导致的。

KeyValueStoreManager 是测试的。DistributedStoreManager 和 KeyColumnValueStoreManager 是两个抽象，我们使用的 cassandra 和 hbase 的 storeManager 都继承自这两个。
这几个 storeManager 就有我们需要的操作数据的方法。

### KeyColumnValueStore & KeyValueStore

KeyValueStore 是测试的，KeyColumnValueStore 是真正的。

```java
KeyColumnValueStore
	KCVSProxy (org.janusgraph.diskstorage.keycolumnvalue)
	    TTLKCVS (org.janusgraph.diskstorage.keycolumnvalue.ttl)
	    ExpectedValueCheckingStore (org.janusgraph.diskstorage.locking.consistentkey)
	    KCVSCache (org.janusgraph.diskstorage.keycolumnvalue.cache)
	        ExpirationKCVSCache (org.janusgraph.diskstorage.keycolumnvalue.cache)
	        NoKCVSCache (org.janusgraph.diskstorage.keycolumnvalue.cache)
	    ReadOnlyKeyColumnValueStore (org.janusgraph.diskstorage.keycolumnvalue)
	BaseKeyColumnValueAdapter (org.janusgraph.diskstorage.keycolumnvalue.keyvalue)
	    OrderedKeyValueStoreAdapter (org.janusgraph.diskstorage.keycolumnvalue.keyvalue)
	CQLKeyColumnValueStore (org.janusgraph.diskstorage.cql)
	HBaseKeyColumnValueStore (org.janusgraph.diskstorage.hbase)
	CassandraEmbeddedKeyColumnValueStore (org.janusgraph.diskstorage.cassandra.embedded)
	CassandraThriftKeyColumnValueStore (org.janusgraph.diskstorage.cassandra.thrift)
	AstyanaxKeyColumnValueStore (org.janusgraph.diskstorage.cassandra.astyanax)
	MetricInstrumentedStore (org.janusgraph.diskstorage.util)
	CounterKCVS in KCVSCacheTest (org.janusgraph.diskstorage.cache)
	InMemoryKeyColumnValueStore (org.janusgraph.diskstorage.keycolumnvalue.inmemory)
```

KeyColumnValueStore 的作用我暂时不是很清楚，从继承类的构造方法看，需要传入一个 StoreManager connection table columnFamily store 。大概能猜出一个 Store 代表一个表格，或者代表一个列族，应该是代表某种数据，例如索引，日志等。

从他的方法可以看出主要是查询库， 如 getKeySlice mutate mutateMany 。

KCVSCache 也继承自 KeyColumnValueStore，名字可以看出是放在内存的 store ，自然也有 getSlice 等方法，我们可以看他的实现类 ExpirationKCVSCache。
这个类里面有一个 Cache<KeySliceQuery,EntryList> cache 的对象，用来缓存查询结果。而 KCVSCache 继承自 KCVSProxy ，这个类则代理 KeyColumnValueStore 对象。其实还有一个 TTLKCVS ，应该是带过期时间的 store

### LogManager

LogManager 的注释：Manager interface for opening {@link Log}s against a particular Log implementation.

KCVSLogManager 实现类的注释：
Implementation of {@link LogManager} against an arbitrary {@link KeyColumnValueStoreManager}. 
Issues {@link Log} instances which wrap around a {@link KeyColumnValueStore}.

可以看出 LogManager 主要是将 通过 KeyColumnValueStoreManager 实现 Log，而 log 则是 围绕 KeyColumnValueStore 。

而我们的log包括三部分： managementLogManager txLogManager userLogManager

### Log

Log 的注释： 
Represents a log that allows content to be added to it in the form of messages and 
to read messages and their content from the log via registered {@link MessageReader}s.

KCVSLog 的注释很长。可以看出主要通过 KeyColumnValueStore 实现。
```
/**
 * Implementation of {@link Log} wrapped around a {@link KeyColumnValueStore}. Each message is written as a column-value pair ({@link Entry})
 * into a timeslice slot. A timeslice slot is uniquely identified by:
 * <ul>
 *     <li>The partition id: On storage backends that are key-ordered, a partition bit width can be configured which configures the number of
 *     first bits that comprise the partition id. On unordered storage backends, this is always 0</li>
 *     <li>A bucket id: The number of parallel buckets that should be maintained is configured by
 *     {@link org.janusgraph.graphdb.configuration.GraphDatabaseConfiguration#LOG_NUM_BUCKETS}. Messages are written to the buckets
 *     in round-robin fashion and each bucket is identified by a bucket id.
 *     Having multiple buckets per timeslice allows for load balancing across multiple keys in the storage backend.</li>
 *     <li>The start time of the timeslice: Each time slice is {@link #TIMESLICE_INTERVAL} microseconds long. And all messages that are added between
 *     start-time and start-time+{@link #TIMESLICE_INTERVAL} end up in the same timeslice. For high throughput logs that might be more messages
 *     than the underlying storage backend can handle per key. In that case, ensure that (2^(partition-bit-width) x (num-buckets) is large enough
 *     to distribute the load.</li>
 * </ul>
 *
 * Each message is uniquely identified by its timestamp, sender id (which uniquely identifies a particular instance of {@link KCVSLogManager}), and the
 * message id (which is auto-incrementing). These three data points comprise the column of a log message. The actual content of the message
 * is written into the value.
 * </p>
 * When {@link MessageReader} are registered, one reader thread per partition id and bucket is created which periodically (as configured) checks for
 * new messages in the storage backend and invokes the reader. </br>
 * Read-markers are maintained (for each partition-id & bucket id combination) under a dedicated key in the same {@link KeyColumnValueStoreManager} as the
 * log messages. The read markers are updated to the current position before each new iteration of reading messages from the log. If the system fails
 * while reading a batch of messages, a subsequently restarted log reader may therefore read messages twice. Hence, {@link MessageReader} implementations
 * should exhibit correct behavior for the (rare) circumstance that messages are read twice.
 *
 * Note: All time values in this class are in microseconds. Hence, there are many cases where milliseconds are converted to microseconds.
 *
 * @author Matthias Broecheler (me@matthiasb.com)
 */
```

英语不好就为难了。
每个消息都由它的时间戳、发件人ID，以及消息ID（它是自动递增的）唯一标识。这三个数据组成包括日志消息的列名。消息的实际内容被写入值中。

### IndexProvider

```java
IndexProvider (org.janusgraph.diskstorage.indexing)
    LuceneIndex (org.janusgraph.diskstorage.lucene)
    TestMockIndexProvider (org.janusgraph.graphdb)
    SolrIndex (org.janusgraph.diskstorage.solr)
    ElasticSearchIndex (org.janusgraph.diskstorage.es)
```

上面的 IndexTransaction 包含了 对 IndexProvider的操作。

### VertexIDAssigner StandardIDPool IDBlock 

负责分配 id ，分配原则我们通过运行 VertexIDAssignerTest 查看。

### Element

我们在操作的过程中有很多的 Vertex Property Edge 等，实际上都继承自一个 Element，继承体系确实有点吓人，这里就不展示了，几个 schema 都这么多东西，我们先分类。

首先我们思考一下，为什么会有这么多。其实 gremin 语法本身定义了一堆schema ，而 janus 也有自己的schema ，两个要进行适配器模式，所以还有一组适配器的schema。所以会比较多？

我们先看一下 gremin 的接口 ,主要有三个，`org.apache.tinkerpop.gremlin.structure`下面：
```java
VertexProperty
Vertex
Edge
```
分别代表了属性，顶点，边，然后 gremin 本身对他们进行了一些实现。然后死 janusgraph 的 `org.janusgraph.core` 包下面，有很多一些接口：
```java
JanusGraphElement
    JanusGraphVertex
        InternalVertex (org.janusgraph.graphdb.internal)
        RelationType (org.janusgraph.core)
        VertexLabel (org.janusgraph.core)
	JanusGraphRelation
		JanusGraphEdge (org.janusgraph.core)
		    AbstractEdge (org.janusgraph.graphdb.relations)
		        CacheEdge (org.janusgraph.graphdb.relations)
		        StandardEdge (org.janusgraph.graphdb.relations)
		JanusGraphVertexProperty (org.janusgraph.core)
		    FulgoraVertexProperty (org.janusgraph.graphdb.olap.computer)
		    AbstractVertexProperty (org.janusgraph.graphdb.relations)
		        StandardVertexProperty (org.janusgraph.graphdb.relations)
		        CacheVertexProperty (org.janusgraph.graphdb.relations)
```

这里展示的并不完整。整个 janus 的schema很复杂。只是大概从注释看出，
在 core 包中，JanusGraphVertex 是顶点，JanusGraphRelation 代表顶点关系，分为属性和边两种 ：JanusGraphVertexProperty 和 JanusGraphEdge。
在 internal 包中，对 core 包的类添加些 janus 特有的方法。

另外在 schema 包中还有 RelationType 和 VertexLabel ，两个都是继承自 JanusGraphVertex ，意思是说 VertexLabel VertexProperty EdgeLabel 都是顶点？？？。
这样就好像明白一点，janus 中的 PropertyKey VertexLabel EdgeLabel 都是以顶点的形式保存起来的。

所以我们看 Edge 类型继承体系比较简单，就是 CacheEdge (org.janusgraph.graphdb.relations) StandardEdge (org.janusgraph.graphdb.relations) 继承自 
AbstractEdge ，然后继承 JanusGraphEdge，Edge。
而 Vertex 继承体系很复杂，除了类似 Edge 的继承体系以外，CacheVertex 还多了 JanusGraphSchemaVertex 这个子类，这个子类还有 RelationTypeVertex 和 VertexLabelVertex 两个子类，
实际上很明显，CacheVertex 的子类 JanusGraphSchemaVertex 代表的就是 graph 的 schema ，也是作为 Vertex 保存的。

这个给别人讲一句话就懂了，但是自己分析可能要好几个小时才能明白。这也是学习和自己研究的不同。

### Index 

索引肯定是数据库的重点，我们到目前没有分析过和所以有关的内容。IndexTransaction 是我们遇到的可能和索引相关的内容了，就从 他开始。
IndexTransaction 中有个 BaseTransaction 的对象用来实现事务，通过 IndexProvider 来产生。我们以 ElasticSearchIndex 为例，可以看看他的方法。

例如 register 方法会创建索引，还有 restore 等操作事务的方法。在 ManagementSystem 的 updateIndex 方法中，定义了各种操作 index 的方法。

Index 类继承了 JanusGraphSchemaElement，主要有两类实现类 JanusGraphIndex 和 RelationTypeIndex 。

JanusGraphIndex 的实现类是 JanusGraphIndexWrapper 。可以通过 JanusGraphManagement#buildIndex(String, Class) 新建 。

RelationTypeIndex 的实现类是 RelationTypeIndexWrapper，可以通过 
JanusGraphManagement#buildEdgeIndex(org.janusgraph.core.EdgeLabel, String, org.apache.tinkerpop.gremlin.structure.Direction, org.apache.tinkerpop.gremlin.process.traversal.Order, org.janusgraph.core.PropertyKey...)
和 JanusGraphManagement#buildPropertyIndex(org.janusgraph.core.PropertyKey, String, org.apache.tinkerpop.gremlin.process.traversal.Order, org.janusgraph.core.PropertyKey...)
两个方法建 RelationTypeIndex。

IndexType 定义所有的 JanusGraphIndex，实现包括 CompositeIndexType 和 MixedIndexType。

IndexType IndexProvider 和 Index 的不同在于，Index 和他的实现类 JanusGraphIndex RelationTypeIndexWrapper 都是继承自 JanusGraphSchemaElement ，和 Vertex 一样，代表的是 janus 中的一个顶点。
IndexType 代表了所以类型 ，IndexProvider 则代表的是和索引相关的操作方法 例如 ElasticSearchIndex SolrIndex LuceneIndex。

### StandardScanner

在 Backend 构造方法最后有一句 new StandardScanner。我们看看这个是干啥用的，主要调用地方是  buildStoreIndexScanJob 这个方法，我们发现这个新建了一个 Job。
buildEdgeScanJob 主要就是在 ManagementSystem 的 updateIndex 方法使用，根据方法名可以看出，这是在遍历数据库的job。

StandardScanner 的重点很明显就是它的内部类 Builder。Builder 内部有一个 ScanJob 的变量，实际上 Builder 就是有个 execute 方法，能够执行 ScanJob ，例如 IndexUpdateJob 和 IndexRepairJob。

这个越看越复杂，还是后续分析吧。
