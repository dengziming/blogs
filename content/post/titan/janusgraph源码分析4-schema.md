---
date: 2018-03-22
title: "janusgraph线上schema过程Debug"
author: "邓子明"
tags:
    - janusgraph
    - 源码
categories:
    - 源码分析
comment: true
---

# 初步调试

## 回顾

首先我们通过 debug 官方的 GraphOfGod 大概进行一个简单的调试，然后我们仔细查看 janusgraph 调用栈，分析了关键类。
这次我们主要看看schema 的建立过程，我们上次分析已经知道，其实 schema也是以Vertex的方式存储在内存和数据库中的。
通过 CacheVertex 的子类 JanusGraphSchemaVertex 实现。JanusGraphSchemaVertex 有两个个子类，
```java

AbstractElement (org.janusgraph.graphdb.internal)
	AbstractVertex (org.janusgraph.graphdb.vertices)
		StandardVertex (org.janusgraph.graphdb.vertices)
			CacheVertex (org.janusgraph.graphdb.vertices)
				JanusGraphSchemaVertex (org.janusgraph.graphdb.types.vertices)
					RelationTypeVertex (org.janusgraph.graphdb.types.vertices)
					    PropertyKeyVertex (org.janusgraph.graphdb.types.vertices)
					    EdgeLabelVertex (org.janusgraph.graphdb.types.vertices)
					VertexLabelVertex (org.janusgraph.graphdb.types)
````

中间省略了一些接口。

所以每次 new 一个 EdgeLabelVertex、VertexLabelVertex、PropertyKeyVertex 的时候，调用栈会非常深。

我们大概查看一下这些类的功能。

### AbstractElement

只有一个属性：private long id; 这个id是唯一的。小于0 的是临时id，事务提交时候会分配大于0的id，等于0的是虚拟的并不存在的，大雨0的是物理persist的。

### InternalVertex

图上没有展示 InternalVertex ，这是一个接口，继承自 JanusGraphVertex 和 InternalElement。凡是带 Internal 的都是比原来的多一个 janus 专属方法的类。
所以 InternalVertex 也是比 JanusGraphVertex 多一些 janus 专属的方法 例如： removeRelation addRelation tx()。
JanusGraphVertex 中则是 janus 和 gremin 都会有的方法 ， 例如 addEdge property label。
InternalVertex 有 query() 等方法，

### AbstractVertex

AbstractVertex 继承自 AbstractElement 和 InternalVertex，
AbstractVertex 比 AbstractElement 的 id 基础上多了一个 StandardJanusGraphTx tx 的属性。也就是多了一个事务空值对象。

### StandardVertex

StandardVertex 继承自 AbstractVertex，多了一个 lifecycle 属性和 volatile AddedRelationsContainer addedRelations 属性。应该是通过缓存空值。

### CacheVertex
CacheVertex 继承自 StandardVertex 。多了一个 queryCache 属性。

### JanusGraphSchemaVertex 

JanusGraphSchemaVertex 就是保存 Schema 的 Vertex ，分为两类 RelationTypeVertex 和 VertexLabelVertex。其中 RelationTypeVertex 分为 PropertyKeyVertex 和 EdgeLabelVertex。

## 预览

schema 操作是通过 ManagementSystem <: JanusGraphManagement 完成的。ManagementSystem 内容很复杂，上次已经大概看了他的方法和属性，这次我们着重看一下方法的实现，首先还是再次浏览一下方法。

### 属性

```java
private static final String CURRENT_INSTANCE_SUFFIX = "(current)";

private final StandardJanusGraph graph;
private final Log sysLog;
private final ManagementLogger mgmtLogger;

private final KCVSConfiguration baseConfig;
private final TransactionalConfiguration transactionalConfig;
private final ModifiableConfiguration modifyConfig;
private final UserModifiableConfiguration userConfig;
private final SchemaCache schemaCache;

private final StandardJanusGraphTx transaction;

private final Set<JanusGraphSchemaVertex> updatedTypes;
private final List<Callable<Boolean>> updatedTypeTriggers;

private final Instant txStartTime;
private boolean graphShutdownRequired;
private boolean isOpen;
```

### 构造方法

基本都是直接赋值。StandardJanusGraph 的 openManagement 方法返回一个 ManagementSystem 。

### Instances 操作

getOpenInstancesInternal 
getOpenInstances
forceCloseInstance

判断正在运行的 instance 。

### commit 和 rollback

commit 方法有四步。
1. 判断 transactionalConfig 是否变化，如果变化，将变化写出。
2. transactionalConfig.commit();
3. transaction.commit();
4. 判断 updatedTypes 是否有更新，进行 expire 操作。

rollback 方法，则很简单。直接调用两个 transaction 的 callback ，然后 close。


### getSchemaElement

这个方法返回一个 JanusGraphSchemaElement ，但是实际上返回的是 RelationTypeIndexWrapper 或者 JanusGraphIndexWrapper ，原因未知，这两个类上一节介绍过，。

### buildRelationTypeIndex

包括 buildPropertyIndex 和 buildEdgeIndex。
步骤都是先 生成对应的 RelationTypeMaker，然后 make，然后调用 addSchemaEdge， 最后调用 updateIndex

### getRelationIndex

得到 RelationType 的 Index。
调用 QueryUtil.getVertices(transaction, BaseKey.SchemaName, JanusGraphSchemaCategory.getRelationTypeName(composedName)) 
然后 return new RelationTypeIndexWrapper((InternalRelationType) v);

### getRelationIndexes

得到所有的  Indexs。

### getGraphIndexDirect

直接调用 transaction.getSchemaVertex(JanusGraphSchemaCategory.GRAPHINDEX.getSchemaName(name));
得到 GraphIndex ，GraphIndex 和 RelationTypeIndex 不一样，一个是基于关系的，前者是基于属性的。

### getGraphIndexes

返回所有的 GraphIndex

### createMixedIndex

调用 JanusGraphSchemaVertex indexVertex = transaction.makeSchemaVertex(JanusGraphSchemaCategory.GRAPHINDEX, indexName, def); 得到 vertex
调用 addSchemaEdge(indexVertex, (JanusGraphSchemaVertex) constraint, TypeDefinitionCategory.INDEX_SCHEMA_CONSTRAINT, null); 添加关系
然后调用 updateSchemaVertex(indexVertex);
最终 new JanusGraphIndexWrapper(indexVertex.asIndexType());

可以看出，这个方法其实只是添加了一个顶点，然后和另一个 constraint 顶点简历了一条关系。

### addIndexKey

给已有的 index 添加一个key，这个应该很复杂，我们先跳过。

### createCompositeIndex

创建 CompositeIndex ，GrpahIndex 分为 CompositeIndex 和 mixedIndex

创建过程也是 addSchemaEdge ， updateSchemaVertex，updateIndex

### InnerClass

很多内部类：

```
IndexBuilder   -- 构建 Index
EmptyIndexJobFuture -- 提交的job
UpdateStatusTrigger -- 更新status的触发器
IndexJobStatus -- job 的 status
IndexIdentifier  --标识
```


### addSchemaEdge

上面好几个方法都会调用 addSchemaEdge updateSchemaVertex updateIndex ，我们看看这三个方法。

addSchemaEdge 是私有方法，应该是在内部会调用的。根据名字可以得出这个方法是添加边，而且添加的是 schema 的边，
我们之前已经知道实际上 schema 都是保存为 vertex，而现在就是给这些 Vertex 添加 Edge，这个边的 EdgeLabel 是 BaseLabel.SchemaDefinitionEdge。
例如 某个 PropertyKey 添加一个 Index ，实际上会有两个 SchemaVertex，然后给他们建立一个关系。
我们可以通过查看方法调用时机，基本上是修改 index 或者 schemaVertex ，一般与 updateSchemaVertex 或者 updateIndex 配合执行。

方法大概步骤就是调用 transaction.addEdge(out, in, BaseLabel.SchemaDefinitionEdge) 得到 Edge，
然后调用 edge.property(BaseKey.SchemaDefinitionDesc.name(), desc);最后返回 edge。

### updateSchemaVertex

就一句话 transaction.updateSchemaVertex(schemaVertex);

### updateIndex

IndexJobFuture updateIndex(Index index, SchemaAction updateAction)

SchemaAction 是一个枚举，包括 REGISTER_INDEX REINDEX ENABLE_INDEX DISABLE_INDEX REMOVE_INDEX 。
IndexJobFuture 代表的是提交了的 job，等待返回结果。

方法步骤：

JanusGraphSchemaVertex schemaVertex = getSchemaVertex(index);

更新 dependentTypes ，实际上就是为了更新 updatedTypes。

根据不同的请求，调用 setStatus setUpdateTrigger setJob 。这个过程很复杂，后面再讲解。

## 编码调试

### ManagementSystem 构造方法
调试整个过程总是很麻烦的，我们只能专注某些部分，首先我们主要看一下 ManagementSystem 的构造过程，和使用细节。

打断点进入构造方法,看这一句代码：`this.transaction = (StandardJanusGraphTx) graph.buildTransaction().disableBatchLoading().start();` 一步一步进入调用栈
```java
this.transaction = (StandardJanusGraphTx) graph.buildTransaction().disableBatchLoading().start();
    return graph.newTransaction(immutable);
        tx.setBackendTransaction(openBackendTransaction(tx));
            return backend.beginTransaction(tx.getConfiguration(), retriever);
```

在 Backend 的 beginTransaction 方法停下来，首先看看 `StoreTransaction tx = storeManagerLocking.beginTransaction(configuration)` 的调用栈：

```
// 这个 ExpectedValueCheckingStoreManager 继承自 KCVSManagerProxy ，它内部有个 KeyColumnValueStoreManager manager 。显然是代理模式，当然也可以认为是装饰模式。
StoreTransaction tx = storeManagerLocking.beginTransaction(configuration);
    StoreTransaction inconsistentTx = manager.beginTransaction(configuration);
        return new CassandraTransaction(config);
    StoreTransaction strongConsistentTx = manager.beginTransaction(consistentTxCfg);
        return new CassandraTransaction(config);
    ExpectedValueCheckingTransaction wrappedTx = new ExpectedValueCheckingTransaction(inconsistentTx, strongConsistentTx, maxReadTime);
    return wrappedTx;
```

可以看出 tx 的大概构造，里面有两个 CassandraTransaction 一个是强一致的，一个是非强一致的。

然后是 `CacheTransaction cacheTx = new CacheTransaction(tx, storeManagerLocking, bufferSize, maxWriteTime, configuration.hasEnabledBatchLoading());`

```java
// CacheTransaction 继承自 StoreTransaction 和 LoggableTransaction ，内部有一个 StoreTransaction 对象，显然也是代理模式或者装饰模式。
CacheTransaction cacheTx = new CacheTransaction(tx, storeManagerLocking, bufferSize, maxWriteTime, configuration.hasEnabledBatchLoading());
    就是一堆赋值。
```

CacheTransaction 内部有一个 StoreTransaction 也就是 上面的 tx， 然后还有一个 StoreManager storeManagerLocking。

然后是 `indexTx.put(entry.getKey(), new IndexTransaction(entry.getValue() indexKeyRetriever.get(entry.getKey()), configuration, maxWriteTime));`

```java
indexTx.put(entry.getKey(), new IndexTransaction(entry.getValue(), indexKeyRetriever.get(entry.getKey()), configuration, maxWriteTime));
    new KeyInformation.IndexRetriever() {...省略代码}
    new IndexTransaction()
        index.beginTransaction(config);
            return new DefaultTransaction(config);
```
backend.beginTransaction(tx.getConfiguration(), retriever); 方法中 有很多 Transaction 对象，包括了 cacheTx indexTx 等。BackendTransaction 的构造方法则比较简单，就是直接赋值。

构造方法讨论到这里，我们可以猜测，ManagementSystem 无论是进行简单 schema 增删改查还是操作索引，
背后都是通过这个 BackendTransaction 完成，而 BackendTransaction 内部又有 cacheTx 和 indexTx 等对象完成。
当然还有一个 transactionalConfig 也有一些任务。

### ManagementSystem getVertexLabels

getVertexLabels 方法返回 Iterable<VertexLabel> ，这里可能是通过 guava 进行封装，所以可能调用栈比较深。

```java
getVertexLabels
    QueryUtil.getVertices(transaction, BaseKey.SchemaCategory, JanusGraphSchemaCategory.VERTEXLABEL)
        tx.query().has(key,Cmp.EQUAL,equalityCondition).vertices();
            1. 
            return new GraphCentricQueryBuilder(this, graph.getIndexSerializer());
            3. 
            return has(key.name(),predicate,condition);
                // 这一步实际上就加了一个 条件，就是 `~T$SchemaCategory = VERTEXLABEL`
                constraints.add(new PredicateCondition<String, JanusGraphElement>(key, predicate, condition));
            3. 
            GraphCentricQuery query = constructQuery(ElementCategory.VERTEX);
                 GraphCentricQuery query = constructQueryWithoutProfile(resultType);
                     省略一大堆复杂代码。
                     return new GraphCentricQuery(resultType, conditions, orders, query, limit);
            // 这里是基于guava实现的懒加载模式的 filter
            Iterables.filter(new QueryProcessor<GraphCentricQuery, JanusGraphElement, JointIndexQuery>(query, tx.elementProcessor), JanusGraphVertex.class);

iterator
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

这里我们就已经知道了，其实这里是构造了一个 GraphCentricQuery 封装所有的查询条件逻辑，然后通过 QueryProcessor 进行处理这个 query，调用 next 的时候会进行查询。

上面我们已经得到了 ResultSetIterator ，接下来我们需要遍历这个 iterator。

```java
iterator.hasNext
    1. next = current
    2. tryToComputeNext()
    ...
        1. hasNext:49, ResultSetIterator (org.janusgraph.graphdb.query)
            return next != null;
            
        2. ResultSetIterator (org.janusgraph.graphdb.query).next:65, 
            1. LimitAdjustingIterator (org.janusgraph.graphdb.query).hasNext:68, 
                SubqueryIterator (org.janusgraph.graphdb.util).hasNext:79, 
            2. LimitAdjustingIterator (org.janusgraph.graphdb.query).next:94, 
                SubqueryIterator (org.janusgraph.graphdb.util).next:90, 

iterator.next()  
   return result.
```

这里的代码比较杂乱，首先是 AbstractIterator 和 Iterators 类，然后是 ResultSetIterator LimitAdjustingIterator SubqueryIterator ，然后还有一个 Stream 类。

AbstractIterator 和 Iterators  是 guava 提供的工具类，AbstractIterator 通过封装一个 Iterator，达到缓存和懒加载的效果。
例如 JDBC 的 ResultSet 如果做成一个 Iterator ，每次调用 next 的时候都会移动一次游标，这样就不能多次判断 hasNext。所以可以用 guava 进行封装。

ResultSetIterator 和 guava 达到的效果类似，通过内部装饰一个 ResultSetIterator 。

LimitAdjustingIterator 通过一个 getNewIterator 得到一个 懒加载 Iterator，其实也是和 guava 类似，只不过你可以认为它只能查看 limit 个元素，当遍历完这 limit 个元素，会重新从 0 开始 next limit次，然后再开始。
说的简单一点，如果一个数组有一千个元素，你的迭代器 limit 是 500，那么你只能得到 500 个元素，想要得到500 - 1000 的元素，要重新查询。类似 mysql 的分页

SubqueryIterator 是代表依次查询的结果。先从 indexCache 查，没有就调用查询，查询结果是一个 List ，得到对应的 iterator 后放在 elementIterator 中。

到这里我们就大概明白了整个查询过程，

### containsVertexLabel

containsVertexLabel 方法判断是否存在，直观的方法是直接调用上面的 getVertexLabels 然后判断一下，但是实际上不是这样。

```
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

containsVertexLabel 会启动一个 transation 通过 name 查询这个 schema 的 typeName 对应的 vertexId。和 getVertexLabels 不太一样。

### makeVertexLabel

```java
mgmt.makeVertexLabel(vType.toString()).make();
    1. makeVertexLabel
        transaction.makeVertexLabel(name);
            StandardVertexLabelMaker maker = new StandardVertexLabelMaker(this);
            maker.name(name);
    2. make
        TypeDefinitionMap def = new TypeDefinitionMap();
        tx.makeSchemaVertex(JanusGraphSchemaCategory.VERTEXLABEL,name,def);
            1. schemaVertex = new VertexLabelVertex(this, IDManager.getTemporaryVertexID(IDManager.VertexIDType.GenericSchemaType,temporaryIds.nextID()), ElementLifeCycle.New);
            2. graph.assignID(schemaVertex, BaseVertexLabel.DEFAULT_VERTEXLABEL);
                ....
                element.setId(elementId);
            3. addProperty(schemaVertex, BaseKey.SchemaName, schemaCategory.getSchemaName(name));
                1. StandardVertexProperty prop = new StandardVertexProperty(IDManager.getTemporaryRelationID(temporaryIds.nextID()), key, (InternalVertex) vertex, normalizedValue, ElementLifeCycle.New);
                2. connectRelation(prop);
                    addedRelations.add(r); 
            4. vertexCache.add(schemaVertex, schemaVertex.longId());
            
```

makeVertexLabel 最关键的一步就是 addedRelations.add(r), 添加关系，这样就能在 commit 的时候写到数据库了。

### commit()
```java
commit
    transactionalConfig.commit();
    transaction.commit();
         1. graph.commit(addedRelations.getAll(), deletedRelations.values(), this); // 这里的两个集合分别是改变的 schema 
             
             
             1. final BackendTransaction schemaMutator = openBackendTransaction(tx); // 打开一个 transaction
             2. commitSummary = prepareCommit(addedRelations,deletedRelations, SCHEMA_FILTER, schemaMutator, tx, acquireLocks);
             3. schemaMutator.commit();
             
             4. commitSummary = prepareCommit(addedRelations,deletedRelations, hasTxIsolation? NO_FILTER : NO_SCHEMA_FILTER, mutator, tx, acquireLocks);
             5. mutator.commit();
                 1. storeTx.commit();
                     1. flushInternal();
                     2. tx.commit();
                 2. itx.commit();
                     1. flushInternal();
                     2. indexTx.commit();
          2. releaseTransaction();
```

其实我这里的注释和官方的不太一样，官方将 graph.commit 分为三部分：

//1. Finalize transaction
//2. Assign JanusGraphVertex IDs
//3. Commit
//3.1 Log transaction (write-ahead log) if enabled
//3.2 Commit schema elements and their associated relations in a separate transaction if backend does not support transactional isolation
//[FAILURE] Exceptions during preparation here cause the entire transaction to fail on transactional systems
//or just the non-system part on others. Nothing has been persisted unless batch-loading

经过我的分析，其实这里分两次 prepareCommit + commit ,是根据底层是否支持事务隔离，如果不支持，先 commit 和 schema 相关的变化，否则 schema 和 data 两边一起提交。

当然这个 wal-log 和 prepareCommit 就大有文章。后续在分析。

### createCompositeIndex

建索引，索引类型是 createCompositeIndex

```java
buildCompositeIndex:650, ManagementSystem$IndexBuilder (org.janusgraph.graphdb.database.management)
    1.checkIndexName:489, ManagementSystem (org.janusgraph.graphdb.database.management)
        getGraphIndex:424, ManagementSystem (org.janusgraph.graphdb.database.management)
            getSchemaVertex:878, StandardJanusGraphTx (org.janusgraph.graphdb.transaction) 
                 // 这里 getSchemaVertex 的内容之前已经讨论过。
    updatedTypes.add((PropertyKeyVertex) key);
    2. transaction.makeSchemaVertex(JanusGraphSchemaCategory.GRAPHINDEX, indexName, def);
        // 这个之前已经说过，我们在简单过一遍
        1. graph.assignID(schemaVertex, BaseVertexLabel.DEFAULT_VERTEXLABEL);
        2. addProperty(schemaVertex, BaseKey.SchemaName, schemaCategory.getSchemaName(name));
    3. addSchemaEdge(indexVertex, keys[i], TypeDefinitionCategory.INDEX_FIELD, paras);
    4. updateIndex(index, SchemaAction.REGISTER_INDEX);
        
```
updatedTypes.add((PropertyKeyVertex) key);
schema 分析主要就是这些，我们还有一些地方没细看，接下来我们把几个复杂的过程分析一下。主要包括查询数据库和 update 索引


# 局部调试

上面的调试过程让我们大概明白了每一步的过程，大概都在做什么，接下来我们要深入一些局部，看一下每一步具体都在做什么。

## 1. makePropertyKey 

我们从简单到复杂，首先看 makePropertyKey，看之前我们大概了解几个相关类。

1. JanusGraphSchemaCategory 
这个是 JanusGraph 的所有 schema 的种类，有 EDGELABEL, PROPERTYKEY, VERTEXLABEL, GRAPHINDEX, TYPE_MODIFIER 五种。

2. PropertyKeyVertex
我们所有的 schema 都是以顶点的形式存在数据库中，所以我们 makePropertyKey 也会创建一个顶点，这个顶点的类型是 PropertyKeyVertex 。
他继承自 RelationTypeVertex，PropertyKey， JanusGraphSchemaVertex，InternalRelationType，RelationType，InternalVertex 等类。
他有 getBaseType getRelationIndexes getKeyIndexes 等方法，

3. BaseKey
BaseKey 和 PropertyKeyVertex 类似，PropertyKeyVertex 是我们定义的 schema，而 BaseKey 则是最基本的key，是 schema 的 ProperyKey, 他们是直接放在内存中的。

4. JanusGraphVertexProperty

JanusGraphVertexProperty 代表一个顶点的 Property，和 JanusGraphEdge 一样继承自 JanusGraphRelation。
当我们给一个 JanusGraph 添加 Property，实际上会创建一条关系，同时返回一个 JanusGraphVertexProperty。

5. InternalRelation

InternalRelation 代表一个关系，实际上就是一条边，在 janus 中，分为 VertexProperty 和 Edge 两种，无论是 Edge 还是 VertexProperty ，都是连接两个顶点。
其中 VertexProperty 是连接一个用户创建的顶点和一个 PropertyKey 顶点，而 Edge 是连接两个 PropertyKey 顶点。

6. ElementCategory

元素种类，有 VERTEX, EDGE, PROPERTY 三种，可以用来判断 index 的种类。

### 进入断点

我们进入断点到： makeSchemaVertex:830, StandardJanusGraphTx (org.janusgraph.graphdb.transaction) 


1. `schemaVertex = new PropertyKeyVertex(this, IDManager.getTemporaryVertexID(IDManager.VertexIDType.UserPropertyKey, temporaryIds.nextID()), ElementLifeCycle.New);`

新建一个代表 PropertyKey 的 Vertex。

2. addProperty(schemaVertex, BaseKey.SchemaName, schemaCategory.getSchemaName(name));

```java

1. vertex = ((InternalVertex) vertex).it();

// 新建一个 VertexProperty 的对象
2. StandardVertexProperty prop = new StandardVertexProperty(IDManager.getTemporaryRelationID(temporaryIds.nextID()), key, (InternalVertex) vertex, normalizedValue, ElementLifeCycle.New);

3.connectRelation(InternalRelation r) 
    
    success = r.getVertex(i).addRelation(r);
        r.getVertex(i) 返回的是前面创建的 PropertyKeyVertex
        addRelation 是在这个 Vertex 内部调用 addedRelations.add(r)
    
    addedRelations.add(r); // 这个 addedRelations 是 StandardJanusGraph 的全局变量

```

我们可以看出，addProperty 实际上就是给 顶点和另一个 PropertyKey 建立一条边。

到这里似乎就完成了，整个过程实际上就是修改了 addedRelations 。

## makeVertexLabel makeEdgeLabel

这两个与 PropertyKey 类似，首先 new JanusGraphSchemaVertex ，分别是 PropertyKeyVertex EdgeLabelVertex VertexLabelVertex 。 然后调用 addProperty 。
addProperty 会 new 一个 StandardVertexProperty ，然后调用 connectRelation(prop) 。将 prop 中的 Relation 都建立连接，添加到 addedRelations。


## commit prepareCommit

//1) Collect deleted edges and their index updates and acquire edge locks
略
//2) Collect added edges and their index updates and acquire edge locks

```java

// 前面所有的关系 关系类型是 InternalRelation ，实现有 StandardVertexProperty 和 StandardEdge 两种
for (InternalRelation add : Iterables.filter(addedRelations,filter)) {
    
    // 每个 Relation 联系多个顶点，如果是 StandardVertexProperty 顶点就是 JanusGraphVertex，如果是 StandardEdge，顶点就是连接的两个 JanusGraphVertex
    for (int pos = 0; pos < add.getLen(); pos++) {
    	// 得到对应的顶点，可能有一个或者两个
    	InternalVertex vertex = add.getVertex(pos);
    	if (pos == 0 || !add.isLoop()) {
    	    
    	    // mutatedProperties: key 是关系连接的 vertex，value 是关系
    	    if (add.isProperty()) mutatedProperties.put(vertex,add);
    	    // mutations: key 是 vertex id, 关系是 add
    	    mutations.put(vertex.longId(), add);
    	}
    	if (!vertex.isNew() && acquireLock(add,pos,acquireLocks)) {
    	    Entry entry = edgeSerializer.writeRelation(add, pos, tx);
    	    mutator.acquireEdgeLock(idManager.getKey(vertex.longId()), entry.getColumn());
        }
    }
    // indexUpdates : IndexSerializer.IndexUpdate
    indexUpdates.addAll(indexSerializer.getIndexUpdates(add));
}
```

//3) Collect all index update for vertices

```java
for (InternalVertex v : mutatedProperties.keySet()) {
    indexUpdates.addAll(indexSerializer.getIndexUpdates(v,mutatedProperties.get(v)));
}
```

//4) Acquire index locks (deletions first)

```java
for (IndexSerializer.IndexUpdate update : indexUpdates) {
    if (!update.isCompositeIndex() || !update.isDeletion()) continue;
    CompositeIndexType iIndex = (CompositeIndexType) update.getIndex();
    if (acquireLock(iIndex,acquireLocks)) {
        mutator.acquireIndexLock((StaticBuffer)update.getKey(), (Entry)update.getEntry());
    }
}
for (IndexSerializer.IndexUpdate update : indexUpdates) {
    if (!update.isCompositeIndex() || !update.isAddition()) continue;
    CompositeIndexType iIndex = (CompositeIndexType) update.getIndex();
    if (acquireLock(iIndex,acquireLocks)) {
        mutator.acquireIndexLock((StaticBuffer)update.getKey(), ((Entry)update.getEntry()).getColumn());
    }
}
```

//5) Add relation mutations
```java
// 遍历 mutations，
for (Long vertexid : mutations.keySet()) {
    Preconditions.checkArgument(vertexid > 0, "Vertex has no id: %s", vertexid);
    List<InternalRelation> edges = mutations.get(vertexid);
    List<Entry> additions = new ArrayList<Entry>(edges.size());
    List<Entry> deletions = new ArrayList<Entry>(Math.max(10, edges.size() / 10));
    
    // 这个顶点所有的 edges
    for (InternalRelation edge : edges) {
        InternalRelationType baseType = (InternalRelationType) edge.getType();
        assert baseType.getBaseType()==null;
        // 这个 InternalRelationType 的所有 type ，这里有点不太懂
        for (InternalRelationType type : baseType.getRelationIndexes()) {
            if (type.getStatus()== SchemaStatus.DISABLED) continue;
            
            // Arity 应该是数据的量，代表的意义应该是 LIST SINGLE 等
            for (int pos = 0; pos < edge.getArity(); pos++) {
                if (!type.isUnidirected(Direction.BOTH) && !type.isUnidirected(EdgeDirection.fromPosition(pos)))
                    continue; //Directionality is not covered
                if (edge.getVertex(pos).longId()==vertexid) {
                    // 序列化数据
                    StaticArrayEntry entry = edgeSerializer.writeRelation(edge, type, pos, tx);
                    if (edge.isRemoved()) {
                        deletions.add(entry);
                    } else {
                        Preconditions.checkArgument(edge.isNew());
                        int ttl = getTTL(edge);
                        if (ttl > 0) {
                            entry.setMetaData(EntryMetaData.TTL, ttl);
                        }
                        // 添加到 additions
                        additions.add(entry);
                    }
                }
            }
        }
    }

    StaticBuffer vertexKey = idManager.getKey(vertexid);
    // 写出数据
    mutator.mutateEdges(vertexKey, additions, deletions);
}
```

//6) Add index updates

```java
for (IndexSerializer.IndexUpdate indexUpdate : indexUpdates) {
    assert indexUpdate.isAddition() || indexUpdate.isDeletion();
    if (indexUpdate.isCompositeIndex()) {
        IndexSerializer.IndexUpdate<StaticBuffer,Entry> update = indexUpdate;
        if (update.isAddition())
            // 直接调用 update 的方法
            mutator.mutateIndex(update.getKey(), Lists.newArrayList(update.getEntry()), KCVSCache.NO_DELETIONS);
        else
            mutator.mutateIndex(update.getKey(), KeyColumnValueStore.NO_ADDITIONS, Lists.newArrayList(update.getEntry()));
    } else {
        IndexSerializer.IndexUpdate<String,IndexEntry> update = indexUpdate;
        has2iMods = true;
        IndexTransaction itx = mutator.getIndexTransaction(update.getIndex().getBackingIndexName());
        String indexStore = ((MixedIndexType)update.getIndex()).getStoreName();
        if (update.isAddition())
            itx.add(indexStore, update.getKey(), update.getEntry(), update.getElement().isNew());
        else
            itx.delete(indexStore,update.getKey(),update.getEntry().field,update.getEntry().value,update.getElement().isRemoved());
    }
}
```

到这里我们可能比较迷惑的就是 IndexUpdate 是怎么获得的。

获得 IndexUpdate 的思路大概是这样：以 CompositeIndex 为例，假如一个顶点，USER，有 name 和 sex 两个 PropertyKey，并且基于 name 和 sex 做了一个 CompositeIndex。
现在有一个顶点，假设 id 为 007，我设置了他的 name 为 "deng"，然后我们需要获得这个用户的 sex ，假设为 "male"，这时候我们需要在 index 插入一条记录 (deng,male) => 007。


所以我们可以看 getIndexUpdates 的源代码，首先是  IndexField[] fields = index.getFieldKeys() 得到这个 index 所有的 filedKey， 然后 new RecordEntry[fields.length]，得到一个
和 fields 长度一样的 RecordEntry 数组，然后从 pos=0 开始给 IndexField 数组赋值，直到 pos >= fields.length。这样就得到了所以和这个属性更新相关的索引更新。
已上面的例子为例，那么得到的 RecordEntry[] 就是 [deng,male]。

然后我们得到了 indexUpdate additions 就是分别将他们写到数据库了。