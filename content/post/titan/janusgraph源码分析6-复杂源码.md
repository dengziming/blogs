---
date: 2018-03-22
title: "janus源码分析5-复杂操作分析"
author: "邓子明"
tags:
    - 图计算
    - janusgraph
categories:
    - 源码分析
comment: true
---

# 源码分析

## 查询操作

之前已经遇到过很多查询操作，比如查询 schema

```java
mgmt.containsVertexLabel(vType.toString())
    transaction.containsVertexLabel(name);
        return getSchemaVertex(JanusGraphSchemaCategory.VERTEXLABEL.getSchemaName(name))!=null;
        1. JanusGraphSchemaCategory.VERTEXLABEL.getSchemaName(name) // 这一步就是在 name 前面加上标识，例如 vl rt
        2. JanusGraphSchemaVertex getSchemaVertex(String schemaName)
            graph.getSchemaCache().getSchemaId(schemaName)
            1. getSchemaCache 
            2. StandardSchemaCache.getSchemaId
                id = retriever.retrieveSchemaByName(schemaName); // 这个 retriever 是 StandardJanusGraph 中的变量 typeCacheRetrieval ，
                    typeCacheRetrieval.retrieveSchemaByName
                        StandardJanusGraph.this.newTransaction
                            QueryUtil.getVertices(consistentTx, BaseKey.SchemaName, typeName)
                            return v!=null?v.longId():null;
        
```

和

```java

iterator()
return new ResultSetIterator(getUnfoldedIterator(),(query.hasLimit()) ? query.getLimit() : Query.NO_LIMIT);   
    1. QueryProcessor (org.janusgraph.graphdb.query).getUnfoldedIterator:107, 
        Iterator<R> subiter = new LimitAdjustingIterator(subq);
    2. this.next = nextInternal();
        hasNext:68, LimitAdjustingIterator (org.janusgraph.graphdb.query)
            getNewIterator:209, QueryProcessor$LimitAdjustingIterator (org.janusgraph.graphdb.query)
                execute:1150, StandardJanusGraphTx$elementProcessorImpl (org.janusgraph.graphdb.transaction)
                    new SubqueryIterator
                        indexCache.getIfPresent(subQuery); // 这里的 schema 应该都是在启动的时候 cache 到了内存中，所以直接得到了，如果是 数据，应该要查询
```

其实查询操作并不复杂，这是有很多层的嵌套，导致我们看起来很麻烦而已，前面我们已经大概介绍了： 首先是 AbstractIterator 和 Iterators 类，然后是 ResultSetIterator LimitAdjustingIterator SubqueryIterator ，然后还有一个 Stream 类。

### Executing vertex centric queries

返回结果是一个 vertex 的 所有 Relation 的 subset。通过 VertexCentricQueryBuilder 构建查询条件。查询会通过条件限制找原始，一共有三种限制：
- Cmp: Comparison constraints (==, !=, <, <=, >, >=, interval: "a <= x < b")
- Geo: Geographic shape constraints (intersect, disjoint, within)
- Text: Text constraints (contains, starts with)

查询的步骤：

- VertexCentricQueryBuilder 构建 VertexCentricQuery，然后将 VertexCentricQuery 传给 QueryProcessor<VertexCentricQuery, JanusGraphRelation> edgeProcessor，
得到所有的 matching records。
- VertexCentricQuery 传给 EdgeSerializer，得到一个 FittedSliceQuery，通过在 edgestore 的 row 中 找到尽量小而且匹配所有满足条件的的 segment，结果返回一个带 byte buffers specifying begin and end of this segment.
- The calculated segment is traversed until enough matching records have been found or the end of the segment is reached. The segment may or may not "fit", that is, it may or may not guarantee that all relations inside the segment are guaranteed to match. If this is not guaranteed, matching is checked explicitly for each visited relation.

我们一层一层进行查看：

### Executing global graph queries

一般步骤：
- ElementQuery 传给 QueryProcessor，得到满足条件的记录。
- ElementQuery 传给 IndexSerializer，如果只有 one equality constraint ，将使用自己的index， 否则使用 external index 。

### VertexCentricQueryBuilder 和 GraphCentricQueryBuilder

GraphCentricQueryBuilder 是用来构造一个 Query 的。它的很多方法都和 gremin 对接，最重要的方法还是 constructQuery ，用来构造 Query。
他的方法 iterables(final GraphCentricQuery query, final Class<E> aClass) 返回一个对应 GraphCentricQuery 结果迭代器。

BasicVertexCentricQueryBuilder 是 VertexCentricQueryBuilder 的父类，StandardJanusGraphTx 的 query(JanusGraphVertex vertex) 会产生一个 VertexCentricQueryBuilder。

继承自 JanusGraphVertexQuery 和 BasicVertexCentricQueryBuilder，用来构建 Query。它主要有 execute 方法。
BasicVertexCentricQueryBuilder 有 constructQuery 方法。


### Query

继承体系：

```java

Query (org.janusgraph.graphdb.query)
	ElementQuery (org.janusgraph.graphdb.query)
	    GraphCentricQuery (org.janusgraph.graphdb.query.graph)
	    VertexCentricQuery (org.janusgraph.graphdb.query.vertex)
	BaseQuery (org.janusgraph.graphdb.query)
	    MultiKeySliceQuery (org.janusgraph.graphdb.query.graph)
	    JointIndexQuery (org.janusgraph.graphdb.query.graph)
	    RawQuery (org.janusgraph.diskstorage.indexing)
	    BaseVertexCentricQuery (org.janusgraph.graphdb.query.vertex)
	        VertexCentricQuery (org.janusgraph.graphdb.query.vertex)
	    SliceQuery (org.janusgraph.diskstorage.keycolumnvalue)
	        KeyRangeQuery (org.janusgraph.diskstorage.keycolumnvalue)
	        KeySliceQuery (org.janusgraph.diskstorage.keycolumnvalue)
	    KVQuery (org.janusgraph.diskstorage.keycolumnvalue.keyvalue)
	    IndexQuery (org.janusgraph.diskstorage.indexing)
	    IndexQueryBuilder (org.janusgraph.graphdb.query.graph)
	    GraphCentricQuery (org.janusgraph.graphdb.query.graph)
	BackendQuery (org.janusgraph.graphdb.query)
	    MultiKeySliceQuery (org.janusgraph.graphdb.query.graph)
	    JointIndexQuery (org.janusgraph.graphdb.query.graph)
	    SliceQuery (org.janusgraph.diskstorage.keycolumnvalue)
	        KeyRangeQuery (org.janusgraph.diskstorage.keycolumnvalue)
	        KeySliceQuery (org.janusgraph.diskstorage.keycolumnvalue)
	    IndexQuery (org.janusgraph.diskstorage.indexing)
	    Subquery in JointIndexQuery (org.janusgraph.graphdb.query.graph)
```

我们主要能发现 BaseQuery 和 BackendQuery 两大子类，

BaseQuery 比较简单，里面就一个 limit 属性，应该是指返回的条数。而 BackendQuery 接口更简单，只有一个方法 updateLimit(int newLimit)，返回一个新的 BackendQuery。至于有什么用后续才能知道。

基于 BaseQuery 和 BackendQuery ，有很多子类。

SliceQuery 有两个 StaticBuffer 类型的属性： sliceStart 和 sliceEnd 。前面说的返回一个带 byte buffers specifying begin and end of this segment.。

KeySliceQuery 继承自 SliceQuery ，扩展 SliceQuery ，增加了 StaticBuffer 类型的 key，能够查询某个 key 的 slice。

KeyRangeQuery 继承自 SliceQuery ，扩展 SliceQuery ，增加了两个 StaticBuffer 类型的 keyStart keyEnd 。为何这样就要查询 bigtable 相关资料了。

MultiKeySliceQuery 继承自 BaseQuery 和 BackendQuery ，内部有一个 List<KeySliceQuery> queries。很明显这是多个 key 一起查。 

IndexQuery 官方注释 在 IndexProvider 中执行的外部 query，query 由两部分组成：一个是查询应该执行的 store 的标识符，另一个是查询的条件。
IndexProvider 的代码我们介绍过，是指外部索引，例如 ElasticSearchIndex ，主要有 register mutate restore query 等方法，很明显是提供一些查询。

JointIndexQuery 的静态内部类 Subquery 继承自 BackendQuery ，内部有两个主要属性：  IndexType index; BackendQuery query; 
Index 可以是 MixedIndexType 或者 CompositeIndexType，对应的 query 分别是 IndexQuery 和 MultiKeySliceQuery
JointIndexQuery 则有 List<Subquery> queries 属性代表很多个 Subquery。
我们可以看出其实 Subquery 代表的是可以在一种索引平台上执行的查询。而 JointIndexQuery 则是很多个这样的查询，可以在各自的平台上进行查询。

GraphCentricQuery 包含了一个 Condition<JanusGraphElement> condition 作为条件，一个 BackendQueryHolder<JointIndexQuery> indexQuery 保存 Query 信息。
BaseVertexCentricQuery 包含了 Condition<JanusGraphRelation> condition 作为添加 ，List<BackendQueryHolder<SliceQuery>> queries 保存 Query 信息
VertexCentricQuery 继承自 BaseVertexCentricQuery ，添加一个 InternalVertex vertex ，至于干啥的还不知道。
他们都是 ElementQuery。

看到这里我们大概能看出 ：
GraphCentricQuery 是基于 JanusGraphElement 的，查询需要 JointIndexQuery ，
JointIndexQuery 内部则是 Subquery，Subquery 主要分为 MixedQuery 和 CompositeQuery，对应的查询分别为 IndexQuery 和 MultiKeySliceQuery，对应的索引分别为 MixedIndexType 和 CompositeIndexType
VertexCentricQuery 是基于 JanusGraphRelation 的，查询需要 SliceQuery ，SliceQuery 就是查询 key + cf 对应的所有的 keyvalue 。

RawQuery 继承自 BaseQuery ，没什么特殊参数，我想应该是值一些粗糙的直接查询。
剩下的 IndexQueryBuilder 和 KVQuery 先不说了。

我们可以看出这些 Query 只是一些描述性的东西，并没有任何执行调用的方法。通过类的关系我们也大概能总结一下：

- 一切都是为了查出 janus 中的元素，所以 是围绕 ElementQuery 展开，ElementQuery 有两个子类，GraphCentricQuery 和 VertexCentricQuery。
GraphCentricQuery 代表以 graph 为中心的查询，例如查询 name=aaa 的所有顶点，VertexCentricQuery 代表以 vertex-centric 的查询，例如查和某个人关系为同事的所有人。
为了完成 GraphCentricQuery 包括两类：IndexQuery 和 MultiKeySliceQuery ，IndexQuery 代表使用外部索引的查询，MultiKeySliceQuery 代表使用 bigtable 自带索引的查询。
这两种合在一起就是 Subquery ，而 JointIndexQuery 内部有多个 Subquery，GraphCentricQuery 中有一个 JointIndexQuery 对象。
为了完成 VertexCentricQuery，也就是加快基于 PropertyKey 和 EdgeLabel 的查询，需要使用 SliceQuery 进行配合。SliceQuery 有很多实现，除了本身还有 KeySliceQuery 和 KeyRangeQuery。
- 而 RawQuery 看名字猜测是直接查询。
- IndexQueryBuilder 就是一个 Builder，内部有一个 IndexSerializer ，它的 execute 方法，实际上就是调用 IndexSerializer 的 executeQuery。




### QueryExecutor

之前我们已经见到介绍过 StandardJanusGraphTx ，实际上这个代表的就是一个事务，内部有很多操作图的方法，我们这次主要是看看他的 elementProcessorImpl 和 edgeProcessorImpl。
他的定义：QueryExecutor<GraphCentricQuery, JanusGraphElement, JointIndexQuery> elementProcessorImpl ，
QueryExecutor<VertexCentricQuery, JanusGraphRelation, SliceQuery> edgeProcessorImpl
听名字就知道大概是执行查询的？这是一个匿名内部类，继承自 QueryExecutor，主要方法是 execute。

#### elementProcessorImpl

我们只看 execute 方法，如果 indexQuery.isEmpty() 会告诉你 "Query requires iterating over all vertices [{}]. For better performance, use indexes"。
最后返回了一个 
new SubqueryIterator(indexQuery.getQuery(0), indexSerializer, txHandle, indexCache, indexQuery.getLimit(), 
getConversionFunction(query.getResultType()),retrievals.isEmpty() ? null: QueryUtil.processIntersectingRetrievals(retrievals, indexQuery.getLimit()));

这里 SubQueryIterator 就是上面讲的。

#### edgeProcessorImpl
他的 execute 方法：
```java

final InternalVertex v = query.getVertex();
final EntryList iterable = v.loadRelations(sq, query1 -> QueryProfiler.profile(profiler, query1, q -> graph.edgeQuery(v.longId(), q, txHandle)));
return RelationConstructor.readRelation(v, iterable, StandardJanusGraphTx.this).iterator();
```
最终会调用 BackendTransation 的  edgeStoreQuery(final KeySliceQuery query)。

### QueryProcessor

Executes a given {@link ElementQuery} against a provided {@link QueryExecutor} to produce the result set of elements.

看名字我们可以认为是查询处理器，他实现了 Iterable 接口，说明它是一个迭代器。
iterator 方法返回 `new ResultSetIterator(getUnfoldedIterator(),(query.hasLimit()) ? query.getLimit() : Query.NO_LIMIT);`
ResultSetIterator 只是类似 guava 的一个封装，通过 nextInternal 方法实现 iterator 提前加载。所以我们可以不管，直接当成 getUnfoldedIterator()。

getUnfoldedIterator 方法看起来比较复杂，但是主要的代码就是 
`Iterator<R> newElements = executor.getNew(query);`, `query.getSubQuery(i);new LimitAdjustingIterator(subquery);`, 以及 `executor.execute(query, backendQuery, executionInfo, profiler)`

executor 的 getNew  方法代表返回符合条件的新元素，execute 方法应该是执行查询。

### IndexSerializer

从 JointIndexQuery 我们能看出，SubQuery 是在 IndexSerializer 中执行的，我们大概了解一下 IndexSerializer。

内部有一个 Map<String, ? extends IndexInformation> mixedIndexes，IndexInformation 有很多子类，例如 ElasticSearchIndex，
还有很多内部类 IndexInfoRetriever IndexRecords IndexUpdate RecordEntry。这应该是一直设计模式吧。
而它的 executeQuery 方法，最终会调用 backendTx.rawQuery(index.getBackingIndexName(), rawQuery) 方法。这里有点奇怪的是为什么只有 MixedIndexType

另外 query 方法 有两种情况，如果是 isCompositeIndex ，会得到 MultiKeySliceQuery 并调用 sq.execute(tx)，如果是 MixedQuery ，调用 tx.indexQuery。
然后都是调用 BackendTransaction 的 indexQuery，CompositeIndex 对应的是 indexQuery(final KeySliceQuery query)，MixedIndex 是 indexQuery(final String index, final IndexQuery query)。
这两个方法将会分别跳转到 KeyColumnValueStore.getSlice(KeySliceQuery query, StoreTransaction txh) 和 IndexProvider.query(IndexQuery query, KeyInformation.IndexRetriever information, BaseTransaction tx) 


### Stream
Stream 是 java 自带的类，目的是实现 lambda 编程，如 map filter reduce 等。java.util.list 调用 stream() 方法就返回一个 Stream 对象。Stream 的部分方法：
peek(Consumer) 方法主要用来调试。类似 map ，但是它返回原对象。例如：

```java
Stream.of("one", "two", "three", "four")
    .filter(e -> e.length() > 3)
    .peek(e -> System.out.println("Filtered value: " + e)) // 打印
    .map(String::toUpperCase)
    .peek(e -> System.out.println("Mapped value: " + e))
    .collect(Collectors.toList());
```

limit(long ) 类似 sql 的 limit。
iterator() 返回一个迭代器。

### SubqueryIterator

根据名字大概可以判断 SubqueryIterator 是一个查询结果迭代器，这里的 Subquery 就是上面我们介绍的，它的成员变量：
```java
private final JointIndexQuery.Subquery subQuery;
private final Cache<JointIndexQuery.Subquery, List<Object>> indexCache;
private Iterator<? extends JanusGraphElement> elementIterator;
private List<Object> currentIds;
private QueryProfiler profiler;
```

SubqueryIterator 的构造方法如下：
```java
// 传入了 subQuery 和 indexSerializer
public SubqueryIterator(JointIndexQuery.Subquery subQuery, IndexSerializer indexSerializer, BackendTransaction tx,
        Cache<JointIndexQuery.Subquery, List<Object>> indexCache, int limit,
        Function<Object, ? extends JanusGraphElement> function, List<Object> otherResults) {
    this.subQuery = subQuery;
    this.indexCache = indexCache;
    // 先从缓存里面取
    final List<Object> cacheResponse = indexCache.getIfPresent(subQuery);
    final Stream<?> stream;
    if (cacheResponse != null) {
        stream = cacheResponse.stream();
    } else {
        try {
            currentIds = new ArrayList<>();
            profiler = QueryProfiler.startProfile(subQuery.getProfiler(), subQuery);
            isTimerRunning = true;
            // 缓存没有就查
            stream = indexSerializer.query(subQuery, tx).peek(r -> currentIds.add(r));
        } catch (final Exception e) {
            throw new JanusGraphException("Could not call index", e.getCause());
        }
    }
    // 生成 elementIterator
    elementIterator = stream.limit(limit).filter(e -> otherResults == null || otherResults.contains(e)).map(function).map(r -> (JanusGraphElement) r).iterator();
}
```


### LimitAdjustingIterator

QueryProcessor$LimitAdjustingIterator 

QueryProcessor 主要有两个属性：

```java
private final Q query;
private final QueryExecutor<Q, R, B> executor;
```
这里的 query 就是上面讲的 query ，一般是 GraphCentricQuery 或者 VertexCentricQuery，executor 就是我们上面讲的 edgeProcessorImpl 和 elementProcessorImpl。
它的 iterator 方法返回一个 ResultSetIterator。
LimitAdjustingIterator 初始化的时候会调用 getNewIterator ，这时候执行 executor.execute(query, backendQuery, executionInfo, profiler)。

和它类似的还有 PreSortingIterator ，加了一个排序 。


### 小结

到这里我们基本搞清楚了整个查询过程。
首先我们的代码的查询会生成 GraphCentricQueryBuilder 或者 BasicVertexCentricQueryBuilder，
然后 我们调用 builder 的查询时会生成 GraphCentricQuery 或者 VertexCentricQuery，并 new QueryProcessor<>(query, tx.elementProcessor)。

QueryProcessor 的 iterator 方法生成一个 ResultSetIterator 封装的 LimitAdjustingIterator ，
LimitAdjustingIterator 的 getNewIterator 会调用 QueryExecutor 的 execute 方法，生成 SubqueryIterator 或者 graph.edgeQuery(v.longId(), q, txHandle) 最终调用 edgeStore 的查询
SubqueryIterator 构造方法会调用 indexSerializer.query(subQuery, tx)，最终调用 edgeStore 或者 IndexProvider 的查询。

以上使我们查看源代码的猜想，要想深入了解还需要进一步 debug 代码。通过网络查到的资料我们进一步进行总结。




















## 更新索引


### Index

Index 类继承自  JanusGraphSchemaElement ，后者我们已经讲过代表 schema 的元素，它的子类如 PropertyKeyVertex 代表 schema 的一部分。
Index 有两个子类 JanusGraphIndex 和 RelationTypeIndex ，分别代表 Graph index 和 基于 Relation 的 Index ，实现类分别是 ：JanusGraphIndexWrapper 和 RelationTypeIndexWrapper。

JanusGraphIndexWrapper 包括了 composite indexes 和 mixed indexes。可以通过 JanusGraphManagement#buildIndex(String, Class) 构造，
通过 JanusGraphManagement#getGraphIndex(String) 或者 JanusGraphManagement#getGraphIndexes(Class) 获得。注意方法包括：

```java
getBackingIndex
getFieldKeys
getIndexedElement
getIndexStatus
getParametersFor
isCompositeIndex
isMixedIndex
isUnique
name
```

RelationTypeIndex 包括 EdgeIndex 和 PropertyKeyIndex ，通过 JanusGraphManagement#buildEdgeIndex(org.janusgraph.core.EdgeLabel ...)和 JanusGraphManagement#buildPropertyIndex(org.janusgraph.core.PropertyKey...) 构造，
通过JanusGraphManagement#getRelationIndex(org.janusgraph.core.RelationType, String) 获得。主要方法包括：

```java
getDirection
getIndexStatus
getSortKey
getSortOrder
getType
```

### IndexType InternalRelationType

JanusGraphIndex 和 RelationTypeIndex 中分别有一个 IndexType 和 InternalRelationType 的属性。

IndexType 又有 CompositeIndexType 和 MixedIndexTypeWrapper 两大子类， CompositeIndexType 还有一个子类是 BaseKey 的索引， 也就是 schema 默认有的索引。

CompositeIndexTypeWrapper 和 MixedIndexTypeWrapper 的构造方法需要传入一个 SchemaSource 对象，也就是 JanusGraphSchemaVertex 的对象。


### IndexBuilder

IndexBuilder 是 JanusGraphManagement 内部接口，顾名思义是用来构建索引的，建造者模式。里面封装了索引的属性，例如： addKey indexOnly unique 等。

实现类在 ManagementSystem 中，实现类 主要属性：
```
private final String indexName;
private final ElementCategory elementCategory;
private boolean unique = false;
private JanusGraphSchemaType constraint = null;
private final Map<PropertyKey, Parameter[]> keys = new HashMap<>();
```

主要方法还是 createCompositeIndex 和 buildMixedIndex 。都会调用宿主类的方法。
实际上创建索引过程就是创建一个 INDEX 类型的 SchemaVertex ，然后建立到 对应的 PropertyKey 的 Edge。

### UpdateStatusTrigger

根据名字判断是更新 status 的触发器。它的属性：
```java
private final StandardJanusGraph graph;
private final long schemaVertexId;
private final SchemaStatus newStatus;
private final Set<Long> propertyKeys;
```

构造方法：
```java
private UpdateStatusTrigger(StandardJanusGraph graph, JanusGraphSchemaVertex vertex, SchemaStatus newStatus, Iterable<PropertyKeyVertex> keys) {
    this.graph = graph;
    this.schemaVertexId = vertex.longId();
    this.newStatus = newStatus;
    this.propertyKeys = Sets.newHashSet(Iterables.transform(keys, new Function<PropertyKey, Long>() {
        @Nullable
        @Override
        public Long apply(@Nullable PropertyKey propertyKey) {
            return propertyKey.longId();
        }
    }));
}
```

call 方法主要就是：
```java
management.setStatus(schemaVertex, newStatus, keys);
management.updatedTypes.addAll(keys);
management.updatedTypes.add(schemaVertex);
management.commit();
```

它被使用的地方是在 updateIndex 的时候，有一步： `setUpdateTrigger(new UpdateStatusTrigger(graph, schemaVertex, SchemaStatus.REGISTERED, keySubset))`
这里的 set 方法只是将它 add 到了一个 List 中，而在调用 commit 的时候，会有个判断，然后调用 `mgmtLogger.sendCacheEviction(updatedTypes, updatedTypeTriggers, getOpenInstancesInternal());`
这里 ManagementLogger 实际上又调用 `evictionTriggerMap.put(evictionId,new EvictionTrigger(evictionId,updatedTypeTriggers,openInstances))` 将它封装为 EvictionTrigger 放进一个 map 中。


这要从新建 StandardJanusGraph 开始说起，在它的构造方法有一句：`mgmtLog.registerReader(ReadMarker.fromNow(), mgmtLogger);`
然后调用 KCVSLog 的 registerReader 方法，然后调用 msgPullers[pos]=new MessagePuller(partitionId,bucketId);
新建 MessagePuller 后，调用 readExecutor.scheduleWithFixedDelay 放进线程池
MessagePuller 的 run 方法会调用 prepareMessageProcessing ，然后调用 readExecutor.submit(new ProcessMessageJob(message,reader)) 放进线程池。
ProcessMessageJob 的 run 方法调用 ManagementLogger 的 read 方法，
然后会调用 EvictionTrigger evictTrigger = evictionTriggerMap.get(evictionId)，这里就取出了我们上面放进去的 evictTrigger，
调用 receivedAcknowledgement 方法，会调用 trigger.call() 方法，然后会 setStatus。

我们稍微总结一下。 StandardJanusGraph 的构造方法实际上会 new 一个 KCVSLog managementLog 和一个 new ManagementLogger managementLogger，前者是日志，后者是 management 的日志。
然后调用 managementLog.registerReader(ReadMarker.fromNow(), managementLogger)，这个 managementLogger 实现了 MessageReader 接口， 也就是将 managementLogger 注册到 KCVSLog 上。
注册以后，会通过一个 ScheduledThreadPoolExecutor 定时调度，将 KCVSLog 按照分区分桶拆分成多个快，发送到 KCVSLog 的消息都会发送给 ManagementLogger。
ManagementLogger 调用 read 方法，判断 MgmtLogType，根据不同的类型，做出不同的响应。当收到 CACHED_TYPE_EVICTION_ACK 类型的消息，将会得到 evictTrigger，并且调用 call 方法。

### StandardScanner

看名字是一个扫描器。内部有 KeyColumnValueStoreManager manager 和  Set<KeyColumnValueStore> openStores ，应该是构造的时候传进来的，来自 graph。
我们比较关心的是他的内部类： Builder ，内部有 ScanJob job，job 有 process 方法，而 Builder 则有 execute 方法，executor 会 new 一个 StandardScannerExecutor，
StandardScannerExecutor executor = new StandardScannerExecutor(job, finishJob, kcvs, storeTx,manager.getFeatures(), numProcessingThreads, workBlockSize, jobConfiguration, graphConfiguration);
executor 是继承自 Runnable 的，然后调用它的 start 方法启动这个线程。executor 的 run 方法就是关键，
StandardScannerExecutor 的 run 方法会 new Processor(job.clone(),processorQueue)，Processor 也是 Runnable ，然后调用 start ，Processor 的 run 中调用了 job 的 process。

这个 job 的 process 方法就是重点。例如 SimpleScanJob 的 process 方法，就是扫描一遍数据库。

StandardScanner 的使用主要是在 updateIndex 的时候，有一步： `builder.setJob(VertexJobConverter.convert(graph, new IndexRepairJob(indexId.indexName, indexId.relationTypeName)));`
这里会设置 job，然后调用 builder.execute()，
里面会 new StandardScannerExecutor，这是一个 Runnable，然后 start。
它的 run 方法会 new Processor(job.clone(),processorQueue) ，这是一个 Runnable ，然后 start。
然后调用  job.process(row.key,row.entries,metrics)。
例如 IndexRepairJob 的 process 方法，会调用 BackendTransaction.mutateIndex 或者 restore 方法，和 IndexSerializer.reindexElement 方法，其实就是重新索引。

想要了解可以在 CassandraScanJobIT 中进行简单测试。


我们可以看出其实  StandardScanner 和 UpdateStatusTrigger 完成工作类似，都是通过线程调用线程，完成所以更新，只不过前者比较简单，后者操作复杂一点。

### ManagementSystem

有关索引的操作也是在 ManagementSystem 中完成，最重要的就是 updateIndex 方法，


### reindex 

mgmt.updateIndex(mgmt.getGraphIndex(indexName), SchemaAction.REINDEX).get();

我们发现这个步骤特别久，就算没有数据也要很久，这不科学。而且打断点也进不去，我们只能直接拍快照，通过分析某个时刻的快照，分析有没有线程死锁的情况。

我们每次在程序运行的时候拍快照都会有两个线程：
```java
"Thread-61@7893" prio=5 tid=0x51 nid=NA waiting
  java.lang.Thread.State: WAITING
	  at sun.misc.Unsafe.park(Unsafe.java:-1)
	  at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:215)
	  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(AbstractQueuedSynchronizer.java:2078)
	  at java.util.concurrent.LinkedBlockingQueue.poll(LinkedBlockingQueue.java:467)
	  at org.janusgraph.diskstorage.keycolumnvalue.scan.StandardScannerExecutor.run(StandardScannerExecutor.java:148)
	  at java.lang.Thread.run(Thread.java:745)

```

```java
"Thread-65@7897" prio=5 tid=0x55 nid=NA waiting
  java.lang.Thread.State: WAITING
	  at sun.misc.Unsafe.park(Unsafe.java:-1)
	  at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:215)
	  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(AbstractQueuedSynchronizer.java:2078)
	  at java.util.concurrent.LinkedBlockingQueue.poll(LinkedBlockingQueue.java:467)
	  at org.janusgraph.diskstorage.keycolumnvalue.scan.StandardScannerExecutor$Processor.run(StandardScannerExecutor.java:272)

```

偶尔还能发现一个：
```java
"Thread-4@4217" daemon prio=5 tid=0x18 nid=NA sleeping
  java.lang.Thread.State: TIMED_WAITING
	  at java.lang.Thread.sleep(Thread.java:-1)
	  at java.lang.Thread.sleep(Thread.java:340)
	  at java.util.concurrent.TimeUnit.sleep(TimeUnit.java:386)
	  at org.janusgraph.diskstorage.util.time.TimestampProviders.sleepPast(TimestampProviders.java:152)
	  at org.janusgraph.graphdb.database.management.ManagementLogger$SendAckOnTxClose.run(ManagementLogger.java:208)
	  at java.lang.Thread.run(Thread.java:745)

```

前两个是常在的线程，在 index 的过程中几乎一致都在，后面那个是偶尔会有出现。

中间还报：

```java
2018-06-30 14:16:35.282 ERROR   --- [      Thread-66] o.j.g.d.management.ManagementLogger      : 
Evicted [23@c0a8007113617-dengzimings-MacBook-Pro-local1] from cache but waiting too long for transactions to close. 
Stale transaction alert on: [standardjanusgraphtx[0x0fd51357], standardjanusgraphtx[0x42d0f747], 
standardjanusgraphtx[0x54168b3c], standardjanusgraphtx[0x27eff5b4], standardjanusgraphtx[0x20cfedd2], 
standardjanusgraphtx[0x7bd7769a], standardjanusgraphtx[0x1095d23a]]
```

这三个可以给我们提供比较多的信息。前面两个可能是由于 poll 的参数等待时间是 100 ms 比较长，所以每次拍快照很大概率刚好在等待。




# debug


