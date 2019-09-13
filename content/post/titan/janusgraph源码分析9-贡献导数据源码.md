---
date: 2018-07-09
title: "janusgraph源码分析8-底层交互"
author: "邓子明"
tags:
    - janusgraph
    - 源码
categories:
    - janusgraph源码分析
comment: true
---

# 反向分析

## cassandra 写数据 API

cassandra 的结构类似 bigtable ，数据实际上是多层嵌套的 map，第一个 key 是 rowkey，第二层key 是 columnFamily，第三层key 是 column，第四层(也可以忽略) 是 timestamp，然后是 value。

写数据的 API 如下：
```java
 CTConnection conn = null;
 try {
     conn = pool.borrowObject(keySpaceName);
     Cassandra.Client client = conn.getClient();
     if (atomicBatch) {
         client.atomic_batch_mutate(batch, consistency);
     } else {
         client.batch_mutate(batch, consistency);
     }
 } catch (Exception ex) {
     throw CassandraThriftKeyColumnValueStore.convertException(ex);
 } finally {
     pool.returnObjectUnsafe(keySpaceName, conn);
 }
```

这里的 batch 就是一个多层嵌套的map。`final Map<ByteBuffer, Map<String, List<org.apache.cassandra.thrift.Mutation>>> batch = new HashMap<>(size);`

这里看起来只有两层，第一层的 ByteBuffer 当然是 rowKey，第二层是 String 是 columnFamily。而 `List<org.apache.cassandra.thrift.Mutation>` 很明显就是添加或者删除的 key:value。

## 写入 cassandra 的数据格式

上面是写 cassandra 的 API，而最终调用这段代码的位置在 `CassandraThriftStoreManager.mutateMany(Map<String, Map<StaticBuffer, KCVMutation>> mutations, StoreTransaction txh)` 方法。

我们需要了解的就是  `Map<String, Map<StaticBuffer, KCVMutation>> mutations` 和 `Map<ByteBuffer, Map<String, List<org.apache.cassandra.thrift.Mutation>>> batch` 的对应关系。

从代码可以看出：

```java
final Map<ByteBuffer, Map<String, List<org.apache.cassandra.thrift.Mutation>>> batch = new HashMap<>(size);

for (final Map.Entry<String, Map<StaticBuffer, KCVMutation>> keyMutation : mutations.entrySet()) {
    
    // mutations 的 key 是 columnFamily
    final String columnFamily = keyMutation.getKey(); 
    
    for (final Map.Entry<StaticBuffer, KCVMutation> mutEntry : keyMutation.getValue().entrySet()) {
        
        // mutations 的第二层 key 是 rowKey
        ByteBuffer keyBB = mutEntry.getKey().asByteBuffer();

        // Get or create the single Cassandra Mutation object responsible for this key
        // Most mutations only modify the edgeStore and indexStore
        
        final Map<String, List<org.apache.cassandra.thrift.Mutation>> cfmutation
            = batch.computeIfAbsent(keyBB, k -> new HashMap<>(3));

        final KCVMutation mutation = mutEntry.getValue();
        final List<org.apache.cassandra.thrift.Mutation> thriftMutation = new ArrayList<>(mutations.size());
        
        // 省略删除的代码。
        
        if (mutation.hasAdditions()) {
            
            for (final Entry ent : mutation.getAdditions()) {
                final ColumnOrSuperColumn columnOrSuperColumn = new ColumnOrSuperColumn();
                
                // mutations 的第三层 key 是 column
                final Column column = new Column(ent.getColumnAs(StaticBuffer.BB_FACTORY));
                // mutations 的 value 是 value
                column.setValue(ent.getValueAs(StaticBuffer.BB_FACTORY));

                column.setTimestamp(commitTime.getAdditionTime(times));

                final Integer ttl = (Integer) ent.getMetaData().get(EntryMetaData.TTL);
                if (null != ttl && ttl > 0) {
                    column.setTtl(ttl);
                }

                columnOrSuperColumn.setColumn(column);
                org.apache.cassandra.thrift.Mutation m = new org.apache.cassandra.thrift.Mutation();
                m.setColumn_or_supercolumn(columnOrSuperColumn);
                thriftMutation.add(m);
            }
        }

        cfmutation.put(columnFamily, thriftMutation);
    }
}
```

我们可以看出 mutateMany 方法的参数和写到 cassandra 的结果不是完全一致，主要是 rowkey 和 columnFamily 的位置是反的。

## 传入 mutateMany 的数据

通过调试可以看出，调用 mutateMany 的地方主要是 `CacheTransation.persist` ,而调用 persist 的就是 flushInternal 方法。相应代码：

```java
// 成员变量： Map<KCVSCache, Map<StaticBuffer, KCVEntryMutation>> mutations

// 新建Map，这个 map 就是上面 mutateMany 的参数，key 分别是 columnFamily 和 rowKey ，
final Map<String, Map<StaticBuffer, KCVMutation>> subMutations = new HashMap<>(mutations.size());

int numSubMutations = 0;
// 遍历 mutations
for (Map.Entry<KCVSCache,Map<StaticBuffer, KCVEntryMutation>> storeMutations : mutations.entrySet()) {
    final Map<StaticBuffer, KCVMutation> sub = new HashMap<>();
    
    // KCVSCache 的 getKey().getName() 就是 columnFamily
    subMutations.put(storeMutations.getKey().getName(),sub);
   
    // mutations 的 value
    for (Map.Entry<StaticBuffer,KCVEntryMutation> mutationsForKey : storeMutations.getValue().entrySet()) {
        if (mutationsForKey.getValue().isEmpty()) continue;
        
        // 将 mutationsForKey 放进去，这个 convert 做了啥没有具体研究，可能只是一个适配。
        sub.put(mutationsForKey.getKey(), convert(mutationsForKey.getValue()));
        numSubMutations+=mutationsForKey.getValue().getTotalMutations();
        if (numSubMutations>= persistChunkSize) {
            numSubMutations = persist(subMutations);
            sub.clear();
            subMutations.put(storeMutations.getKey().getName(),sub);
        }
    }
}
```

## mutations 的构造

上面我们看出了，其实基本上没复杂处理，接下来我们看看 mutations 数据哪里来的。

对于 mutations 的修改操作，来自于 mutate 方法，代码：

```java
// 传入的是 store（包含了columnFamily） key（rowKey） additions 和 deletions
void mutate(KCVSCache store, StaticBuffer key, List<Entry> additions, List<Entry> deletions) throws BackendException {
    Preconditions.checkNotNull(store);
    if (additions.isEmpty() && deletions.isEmpty()) return;
    
    // 构造 KCVEntryMutation
    KCVEntryMutation m = new KCVEntryMutation(additions, deletions);
    
    // 这几步就是简单的合并所以的 additions 和 deletions
    final Map<StaticBuffer, KCVEntryMutation> storeMutation = mutations.computeIfAbsent(store, k -> new HashMap<>());
    KCVEntryMutation existingM = storeMutation.get(key);
    
    if (existingM != null) {
        existingM.merge(m);
    } else {
        storeMutation.put(key, m);
    }

    numMutations += m.getTotalMutations();

    if (batchLoading && numMutations >= persistChunkSize) {
        flushInternal();
    }
}
```

## mutate 方法参数来源

mutate 方法传入的是 store（包含了columnFamily） key（rowKey） additions 和 deletions，这几个参数哪里来的呢？ KCVSCache 的 mutateEntries，
mutateEdges 调用时机呢？ edgeStore.mutateEntries(key, additions, deletions, storeTx); indexStore.mutateEntries(key, additions, deletions, storeTx);
我们先以 edgeStore 为例，在 StandardJanusGraph 的 prepareCommit 方法中，调用了 mutator.mutateEdges(vertexKey, additions, deletions);
代码如下，我们删掉了部分代码，包括 索引和数据删除。
```java
ListMultimap<Long, InternalRelation> mutations = ArrayListMultimap.create();
ListMultimap<InternalVertex, InternalRelation> mutatedProperties = ArrayListMultimap.create();
List<IndexSerializer.IndexUpdate> indexUpdates = Lists.newArrayList();


//2) Collect added edges and their index updates and acquire edge locks
// add 是 InternalRelation ，包括 VertexProperty 和 Edge，前面分析过，VertexProperty 实际上就是顶点和一个 schema 的订单建一条边，Edge 就是两个顶点建一条边。
for (InternalRelation add : Iterables.filter(addedRelations,filter)) {
    Preconditions.checkArgument(add.isNew());
    
    // getLen 返回这个 Relation 的长度，如果是 VertexProperty 是1，Edge 是需要根据方向进行判断
    for (int pos = 0; pos < add.getLen(); pos++) {
        // 得到对应的 vertex 
        InternalVertex vertex = add.getVertex(pos);
        if (pos == 0 || !add.isLoop()) {
        
            // mutatedProperties 的 key: InternalVertex,value:InternalRelation,mutatedProperties 是用于更新索引的，在我们这里实际上没什么用。
            if (add.isProperty()) mutatedProperties.put(vertex,add);
            // mutations 的 key ： vertexId, value ： InternalRelation
            mutations.put(vertex.longId(), add);
        }
        if (!vertex.isNew() && acquireLock(add,pos,acquireLocks)) {
            Entry entry = edgeSerializer.writeRelation(add, pos, tx);
            mutator.acquireEdgeLock(idManager.getKey(vertex.longId()), entry.getColumn());
        }
    }
}


//5) Add relation mutations
for (Long vertexId : mutations.keySet()) {
    Preconditions.checkArgument(vertexId > 0, "Vertex has no id: %s", vertexId);
    final List<InternalRelation> edges = mutations.get(vertexId);
    final List<Entry> additions = new ArrayList<>(edges.size());
    final List<Entry> deletions = new ArrayList<>(Math.max(10, edges.size() / 10));
    for (final InternalRelation edge : edges) {
        // 得到 InternalRelationType ，分为 PropertyKey 和 EdgeLabel 两类
        final InternalRelationType baseType = (InternalRelationType) edge.getType();
        assert baseType.getBaseType()==null;

        for (InternalRelationType type : baseType.getRelationIndexes()) { 
            if (type.getStatus()== SchemaStatus.DISABLED) continue;
            // getArity 和 getLen 不一样，
            for (int pos = 0; pos < edge.getArity(); pos++) {
                if (!type.isUnidirected(Direction.BOTH) && !type.isUnidirected(EdgeDirection.fromPosition(pos)))
                    continue; //Directionality is not covered
                
                // 如果是起始顶点
                if (edge.getVertex(pos).longId()==vertexId) {
                
                    // 根据 edge type pos tx 得到应该序列化的 StaticArrayEntry
                    StaticArrayEntry entry = edgeSerializer.writeRelation(edge, type, pos, tx);
                    if (edge.isRemoved()) {
                        deletions.add(entry);
                    } else {
                        Preconditions.checkArgument(edge.isNew());
                        int ttl = getTTL(edge);
                        if (ttl > 0) {
                            entry.setMetaData(EntryMetaData.TTL, ttl);
                        }
                        additions.add(entry);
                    }
                }
            }
        }
    }

    StaticBuffer vertexKey = idManager.getKey(vertexId);
    mutator.mutateEdges(vertexKey, additions, deletions);
}

```

## edgeSerializer.writeRelation 到底做了什么

我们现在就想知道，数据是怎么被序列化话 entry 的，代码如下：

```java
public StaticArrayEntry writeRelation(InternalRelation relation, InternalRelationType type, int position,
                                      TypeInspector tx) {
    assert type==relation.getType() || (type.getBaseType() != null
            && type.getBaseType().equals(relation.getType()));
    // 得到方向，pos 是 0 就是 out，是 1 就是 in
    Direction dir = EdgeDirection.fromPosition(position);
    
    // 方向验证
    Preconditions.checkArgument(type.isUnidirected(Direction.BOTH) || type.isUnidirected(dir));
    
    // 得到 typeId， 这个 type 是 VertexLabel 或者 PropertyKey
    long typeId = type.longId();
    // 得到 dirID 
    DirectionID dirID = getDirID(dir, relation.isProperty() ? RelationCategory.PROPERTY : RelationCategory.EDGE);
    
    // 得到 一个 out
    DataOutput out = serializer.getDataOutput(DEFAULT_CAPACITY);
    // key 和 value 的分割地址。
    int valuePosition;
    
    // 写 typeId 和 dirID 
    IDHandler.writeRelationType(out, typeId, dirID, type.isInvisibleType());
    
    // 得到 multiplicity 和 sortKey
    Multiplicity multiplicity = type.multiplicity();
    long[] sortKey = type.getSortKey();
    
    assert !multiplicity.isConstrained() || sortKey.length==0: type.name();
    int keyStartPos = out.getPosition();
    if (!multiplicity.isConstrained()) {
        // 如果 multiplicity 是 没有限制，也就是为 MULTI，必须要有 sortKey，写出 sortKey。
        writeInlineTypes(sortKey, relation, out, tx, InlineType.KEY);
    }
    
    // 到这里 key 就写完了，得到 key 的 pos
    int keyEndPos = out.getPosition();

    long relationId = relation.longId();

    //How multiplicity is handled for edges and properties is slightly different
    if (relation.isEdge()) {
        // 得到另一个 vertex 的 id
        long otherVertexId = relation.getVertex((position + 1) % 2).longId();
        // 如果 multiplicity 有限制
        if (multiplicity.isConstrained()) {
            // isUnique
            if (multiplicity.isUnique(dir)) {
                // 得到 valuePosition ，写出 otherVertexId 
                valuePosition = out.getPosition();
                VariableLong.writePositive(out, otherVertexId);
            } else {
                // 反方向写 otherVertexId ,记下 valuePosition
                VariableLong.writePositiveBackward(out, otherVertexId);
                valuePosition = out.getPosition();
            }
            // 写下 relationId
            VariableLong.writePositive(out, relationId);
        } else {
            // 没有限制，反方向写出 otherVertexId 和 relationId ，记下 valuePosition
            VariableLong.writePositiveBackward(out, otherVertexId);
            VariableLong.writePositiveBackward(out, relationId);
            valuePosition = out.getPosition();
        }
    } else { // PropertyKey
        assert relation.isProperty();
        Preconditions.checkArgument(relation.isProperty());
        // 得到 property 的值。
        Object value = ((JanusGraphVertexProperty) relation).value();
        Preconditions.checkNotNull(value);
        PropertyKey key = (PropertyKey) type;
        assert key.dataType().isInstance(value);
        
        // 写出 value 得到 valuePosition
        if (multiplicity.isConstrained()) { // 没有限制的 property
            if (multiplicity.isUnique(dir)) { //Cardinality=SINGLE
                valuePosition = out.getPosition();
                writePropertyValue(out,key,value);
            } else { //Cardinality=SET
                writePropertyValue(out,key,value);
                valuePosition = out.getPosition();
            }
            VariableLong.writePositive(out, relationId);
        } else {
            assert multiplicity.getCardinality()== Cardinality.LIST;
            VariableLong.writePositiveBackward(out, relationId);
            valuePosition = out.getPosition();
            writePropertyValue(out,key,value);
        }
    }

    //Write signature
    long[] signature = type.getSignature();
    writeInlineTypes(signature, relation, out, tx, InlineType.SIGNATURE);

    //Write remaining properties
    LongSet writtenTypes = new LongHashSet(sortKey.length + signature.length);
    if (sortKey.length > 0 || signature.length > 0) {
        for (long id : sortKey) writtenTypes.add(id);
        for (long id : signature) writtenTypes.add(id);
    }
    LongArrayList remainingTypes = new LongArrayList(8);
    for (PropertyKey t : relation.getPropertyKeysDirect()) {
        if (!(t instanceof ImplicitKey) && !writtenTypes.contains(t.longId())) {
            remainingTypes.add(t.longId());
        }
    }
    //Sort types before writing to ensure that value is always written the same way
    long[] remaining = remainingTypes.toArray();
    Arrays.sort(remaining);
    for (long tid : remaining) {
        PropertyKey t = tx.getExistingPropertyKey(tid);
        writeInline(out, t, relation.getValueDirect(t), InlineType.NORMAL);
    }
    assert valuePosition>0;

    return new StaticArrayEntry(type.getSortOrder() == Order.DESC ?
                                out.getStaticBufferFlipBytes(keyStartPos, keyEndPos) :
                                out.getStaticBuffer(), valuePosition);
}
```

其实就是不断吧值写进去并且记录一下值的位置。

我们需要了解一下 writeInline 方法以及  StaticArrayEntry VariableLong 类。

### StaticArrayEntry

```
Entry (org.janusgraph.diskstorage)
BaseStaticArrayEntry (org.janusgraph.diskstorage.util)
StaticEntry in StaticArrayEntryList (org.janusgraph.diskstorage.util)
StaticArrayEntry (org.janusgraph.diskstorage.util)
SwappingEntry in StaticArrayEntryList (org.janusgraph.diskstorage.util)
StaticArrayEntry (org.janusgraph.diskstorage.util)
```

Entry 代表存储在 cassandra 基本结构，有 getColumn getValuePosition getValue 等方法，
BaseStaticArrayEntry 则是利用一个 array,offset,limit,valuePosition 进行封装。

### VariableLong
这个提供了一个读写Long类型的数据的方法，具体后续研究。

### writeInline

writeInline 方法实现：

```java
private void writeInlineTypes(long[] keyIds, InternalRelation relation, DataOutput out, TypeInspector tx,
                              InlineType inlineType) {
    for (long keyId : keyIds) {
        PropertyKey t = tx.getExistingPropertyKey(keyId);
        writeInline(out, t, relation.getValueDirect(t), inlineType);
    }
}

private void writeInline(DataOutput out, PropertyKey inlineKey, Object value, InlineType inlineType) {
    assert inlineType.writeInlineKey() || !AttributeUtil.hasGenericDataType(inlineKey);

    if (inlineType.writeInlineKey()) {
        IDHandler.writeInlineRelationType(out, inlineKey.longId());
    }

    writePropertyValue(out,inlineKey,value, inlineType);
}

private void writePropertyValue(DataOutput out, PropertyKey key, Object value, InlineType inlineType) {
    if (AttributeUtil.hasGenericDataType(key)) {
        assert !inlineType.writeByteOrdered();
        out.writeClassAndObject(value);
    } else {
        assert value==null || value.getClass().equals(key.dataType());
        if (inlineType.writeByteOrdered()) out.writeObjectByteOrder(value, key.dataType());
        else out.writeObject(value, key.dataType());
    }
}
```
从代码我们可以看出，实际上都是对id进行的操作，所以如果知道了顶点的 id，给顶点添加边和属性，实际上不需要查询这个顶点，直接操作即可，所以这给我们导数据提供了一种思路，可以直接操作 id。

## writeRelation 方法的参数怎么构造的

从上面我们可以看出来 edgeSerializer.writeRelation 方法的参数是 (edge, type, pos, tx)，而 edge 来自于对 mutations 的处理，mutations 来自 add ,add 来自 addedRelations。

addedRelations 进行 add 操作的步骤在 StandardJanusGraph 的 connectRelation(InternalRelation r) 方法中。connectRelation 方法有两处调用 addEdge 和 addProperty。

addEdge 和 addProperty 的调用栈就比较多了。

# 正向理清思路

## 1. janus 官网介绍

https://docs.janusgraph.org/latest/schema.html

### Edge Label

Multiplicity 

MULTI SIMPLE MANY2ONE ONE2MANY ONE2ONE 五种，每种的意义可以参考官网。默认的是 MULTI

### Property Keys

dataType(Class) 确定数据类型，Object.class 能够传入任何参数，但是不鼓励。

Property Key Cardinality

SINGLE: Allows at most one value per element for such key. In other words, the key→value mapping is unique for all elements in the graph. The property key birthDate is an example with SINGLE cardinality since each person has exactly one birth date.
LIST: Allows an arbitrary number of values per element for such key. In other words, the key is associated with a list of values allowing duplicate values. Assuming we model sensors as vertices in a graph, the property key sensorReading is an example with LIST cardinality to allow lots of (potentially duplicate) sensor readings to be recorded.
SET: Allows multiple values but no duplicate values per element for such key. In other words, the key is associated with a set of values. The property key name has SET cardinality if we want to capture all names of an individual (including nick name, maiden name, etc).

### Relation Types

Edge labels and property keys are jointly referred to as relation types ,must unique

### Vertex Labels

call makeVertexLabel(String).make() 

### Unidirected Edges

单向的边是只能在向外方向上遍历的边。单指向边具有较低的存储占用，但在它们支持的遍历类型中受到限制。单向的边在概念上类似于万维网中的超链接，在这个意义上，外顶点可以遍历边缘，但是顶点不知道它的存在。

## 2. addProperty 和 addEdge

```java
public JanusGraphVertexProperty addProperty(VertexProperty.Cardinality cardinality, JanusGraphVertex vertex, PropertyKey key, Object value) {
    if (key.cardinality().convert()!=cardinality && cardinality!=VertexProperty.Cardinality.single)
        throw new SchemaViolationException("Key is defined for %s cardinality which conflicts with specified: %s",key.cardinality(),cardinality);
    verifyWriteAccess(vertex);
    Preconditions.checkArgument(!(key instanceof ImplicitKey),"Cannot create a property of implicit type: %s",key.name());
    vertex = ((InternalVertex) vertex).it();
    Preconditions.checkNotNull(key);
    checkPropertyConstraintForVertexOrCreatePropertyConstraint(vertex, key);
    final Object normalizedValue = verifyAttribute(key, value);
    
    // 得到 Cardinality SINGLE LIST SET ，一般是 SINGLE
    Cardinality keyCardinality = key.cardinality();
    
    // 省略部分代码
    try {
          // 省略检查
          
        StandardVertexProperty prop = new StandardVertexProperty(IDManager.getTemporaryRelationID(temporaryIds.nextID()), key, (InternalVertex) vertex, normalizedValue, ElementLifeCycle.New);
        if (config.hasAssignIDsImmediately()) graph.assignID(prop);
        connectRelation(prop);
        return prop;
    } finally {
        uniqueLock.unlock();
    }

}

public JanusGraphEdge addEdge(JanusGraphVertex outVertex, JanusGraphVertex inVertex, EdgeLabel label) {
    verifyWriteAccess(outVertex, inVertex);
    outVertex = ((InternalVertex) outVertex).it();
    inVertex = ((InternalVertex) inVertex).it();
    Preconditions.checkNotNull(label);
    checkConnectionConstraintOrCreateConnectionConstraint(outVertex, inVertex, label);
    Multiplicity multiplicity = label.multiplicity();
    TransactionLock uniqueLock = getUniquenessLock(outVertex, (InternalRelationType) label,inVertex);
    uniqueLock.lock(LOCK_TIMEOUT);
    try {
     // 省略检查
        StandardEdge edge = new StandardEdge(IDManager.getTemporaryRelationID(temporaryIds.nextID()), label, (InternalVertex) outVertex, (InternalVertex) inVertex, ElementLifeCycle.New);
        if (config.hasAssignIDsImmediately()) graph.assignID(edge);
        connectRelation(edge);
        return edge;
    } finally {
        uniqueLock.unlock();
    }
}
```

其实这两个做的最主要的就两步： new StandardEdge new StandardVertexProperty  connectRelation(edge);

connectRelation 最主要的就是 addedRelations.add(r)

最后在 commit 的时候，会 处理  addedRelations，代码逻辑在上面我们已经看过了。我们在简化一下：
```java
for (InternalRelation add : Iterables.filter(addedRelations,filter)) {
    Preconditions.checkArgument(add.isNew());

    for (int pos = 0; pos < add.getLen(); pos++) {
        InternalVertex vertex = add.getVertex(pos);
        if (pos == 0 || !add.isLoop()) {
            // 添加 mutations
            mutations.put(vertex.longId(), add);
        }
    }
}

// 
for (Long vertexId : mutations.keySet()) {

    final List<InternalRelation> edges = mutations.get(vertexId);
    final List<Entry> additions = new ArrayList<>(edges.size());
    final List<Entry> deletions = new ArrayList<>(Math.max(10, edges.size() / 10));
    
    for (final InternalRelation edge : edges) {
        final InternalRelationType baseType = (InternalRelationType) edge.getType();
        assert baseType.getBaseType()==null;

        for (InternalRelationType type : baseType.getRelationIndexes()) { // getRelationIndexes 这里是得到了 RelationTypeIndex 相关的 关系
            if (type.getStatus()== SchemaStatus.DISABLED) continue;
            for (int pos = 0; pos < edge.getArity(); pos++) {
                if (!type.isUnidirected(Direction.BOTH) && !type.isUnidirected(EdgeDirection.fromPosition(pos)))
                    continue; //Directionality is not covered
                if (edge.getVertex(pos).longId()==vertexId) {
                    StaticArrayEntry entry = edgeSerializer.writeRelation(edge, type, pos, tx);
                    if (edge.isRemoved()) {
                        deletions.add(entry);
                    } else {
                        Preconditions.checkArgument(edge.isNew());
                        int ttl = getTTL(edge);
                        if (ttl > 0) {
                            entry.setMetaData(EntryMetaData.TTL, ttl);
                        }
                        additions.add(entry);
                    }
                }
            }
        }
    }

    StaticBuffer vertexKey = idManager.getKey(vertexId);
    mutator.mutateEdges(vertexKey, additions, deletions);
}
```

## mutateEdges 

会逐步调用 
edgeStore.mutateEntries(key, additions, deletions, storeTx);
mutateEntries(StaticBuffer key, List<Entry> additions, List<Entry> deletions, StoreTransaction txh)
mutate

mutate 会将改变都记录到 mutations 中，在 flushInternal 的时候 mutations 会变换一下记录到 subMutations ，然后调用 persist(subMutations);
紧接着调用 manager.mutateMany(subMutations, tx); 最后重构成 cfmutation，通过 cassandra 的 CTConnection 保存到 cassandra 中。

这样看来，整个过程就清晰了。

## id 分配

StandardIDPool 进行 id 的分配，调用 graph.assignID(schemaVertex, BaseVertexLabel.DEFAULT_VERTEXLABEL) 等方法的时候，会调用。

```java
assignID:455, StandardJanusGraph (org.janusgraph.graphdb.database)
assignID:153, VertexIDAssigner (org.janusgraph.graphdb.database.idassigner)
assignID:182, VertexIDAssigner (org.janusgraph.graphdb.database.idassigner)
assignID:308, VertexIDAssigner (org.janusgraph.graphdb.database.idassigner)
nextID:204, StandardIDPool (org.janusgraph.graphdb.database.idassigner)
nextBlock:173, StandardIDPool (org.janusgraph.graphdb.database.idassigner)
startIDBlockGetter:247, StandardIDPool (org.janusgraph.graphdb.database.idassigner)

call:288, StandardIDPool$IDBlockGetter (org.janusgraph.graphdb.database.idassigner)
getIDBlock:213, ConsistentKeyIDAuthority (org.janusgraph.diskstorage.idmanagement)
    getBlockApplication:373, ConsistentKeyIDAuthority (org.janusgraph.diskstorage.idmanagement)
idStore.mutate(partitionKey, Arrays.asList(StaticArrayEntry.of(finalTarget)), KeyColumnValueStore.NO_DELETIONS, txh);
```

这里涉及到了很多东西，而且是在两个线程中完成的，就不太方便处理了。

### VertexIDAssigner

首先是 VertexIDAssigner 的创建：
```java
public VertexIDAssigner(Configuration config, IDAuthority idAuthority, StoreFeatures idAuthFeatures) {
    Preconditions.checkNotNull(idAuthority);
    this.idAuthority = idAuthority;

    int partitionBits = NumberUtil.getPowerOf2(config.get(CLUSTER_MAX_PARTITIONS));
    idManager = new IDManager(partitionBits);
    Preconditions.checkArgument(idManager.getPartitionBound() <= Integer.MAX_VALUE && idManager.getPartitionBound()>0);
    this.partitionIdBound = (int)idManager.getPartitionBound();
    hasLocalPartitions = idAuthFeatures.hasLocalKeyPartition();

    placementStrategy = Backend.getImplementationClass(config, config.get(PLACEMENT_STRATEGY),
            REGISTERED_PLACEMENT_STRATEGIES);
    placementStrategy.injectIDManager(idManager);
    log.debug("Partition IDs? [{}], Local Partitions? [{}]",true,hasLocalPartitions);

    long baseBlockSize = config.get(IDS_BLOCK_SIZE);
    idAuthority.setIDBlockSizer(new SimpleVertexIDBlockSizer(baseBlockSize));

    renewTimeoutMS = config.get(IDS_RENEW_TIMEOUT);
    renewBufferPercentage = config.get(IDS_RENEW_BUFFER_PERCENTAGE);

    idPools = new ConcurrentHashMap<Integer, PartitionIDPool>(partitionIdBound);
    schemaIdPool = new StandardIDPool(idAuthority, IDManager.SCHEMA_PARTITION, PoolType.SCHEMA.getIDNamespace(),
            IDManager.getSchemaCountBound(), renewTimeoutMS, renewBufferPercentage);
    partitionVertexIdPool = new StandardIDPool(idAuthority, IDManager.PARTITIONED_VERTEX_PARTITION, PoolType.PARTITIONED_VERTEX.getIDNamespace(),
            PoolType.PARTITIONED_VERTEX.getCountBound(idManager), renewTimeoutMS, renewBufferPercentage);
    setLocalPartitions(partitionBits);
}
```

里面主要有 idAuthority , idManager(partitionBits=5) partitionIdBound=32  placementStrategy idPools schemaIdPool partitionVertexIdPool .

#### assignID

id 有三个部分组成 [0 count suffix partitionId],count，最高位是0，然后是后缀。后缀在 IDManager 中有配置,partitionId 默认是5位.
第一部是 得到 partitionID ，分为很多种情况，例如 schema 为0，分区的为 -1，vertex 的为 placementStrategy 随机获得。例如8， Relation 通过 incident 获得。


然后才是 assignID
先得到 count ，得到过程是：
通过 partition 在 idPools 得到 PartitionIDPool，如果没有，新建 PartitionIDPool ，然后在每个 PartitionIDPool 中新建 3个 StandardIDPool ，分别对应 NORMAL_VERTEX, UNMODIFIABLE_VERTEX, RELATION;
在 PartitionIDPool 中得到现在的 element 所对应的 idPool，然后调用 count = idPool.nextID()  得到count ,nextID 会调用 currentBlock 得到 id。如果当前的 currentBlock 分配完了，重新申请一个 block

调用 getId 方法的时候，会有一个 uniqueIDBitWidth ，默认是 4位，然后还有一个 unique 数值是0，最后返回的是得到的count 左右4位，如果是1，就是16.
返回了 count，然后构造的 结果就是 00000 0 000010000000

### StandardIDPool

构造传入了：idAuthority partition idNamespace idUpperBound renewBufferPercentage 
还有一个 exec ，用来执行线程。
还有 currentBlock currentIndex renewBlockIndex 记录当前的状态。

###


我们从 assignID 开始看。分为两步：

partitionID = placementStrategy.getPartition(element);


## 调试一次

接下来我们可以调试一次，通过调试过每一步，熟悉每一步的内容。


# bulk loading

接下来我们要做一个导数据的工具。我们有一堆给定好的书籍，然后我们能够将数据导入到 janus 中，我们需要结合 cassandra 和 hbase 自带的 bulk loading 工具。
首先我们需要得到所有的序列化的数据，实际上就是 edge 数据，而 index 的数据我们可以后续调用 reindex。

## vertex 导入

我们可以想象一下，导入边的流程，首先要有一个表格，并且这个表格要带有表头，然后下面的就是数据。表头包括字段名和数据类型，其中第一个是主键。例如有电话的数据，表头结构为：
phone:string,name:string,relation:integer

我们程序首先是验证数据，验证数据主要是重复性检验，格式检验。然后需要创建 schema。读取所有的表头，并创建好 schema。然后导入数据。



