---
date: 2018-07-09
title: "janusgraph源码分析8-索引存储"
author: "邓子明"
tags:
    - janusgraph
    - 源码
categories:
    - janusgraph源码分析
comment: true
---

上一节我们了解了 JanusGraph 的关系存储，主要是在 EdgeSerializer 中的序列化和反序列化，我们还要这次看看 IndexSerializer 的相关类。


# 基础类

## IndexSerializer

用来序列化，反序列化

## IndexProvider

IndexProvider 继承自 IndexInformation ，IndexInformation 主要判断是否支持某个 KeyInformation，
IndexProvider 最主要的是 mutate 方法，该方法就是用来保存数据到底层存储系统。


## KeyInformation

保存key的信息，有三个内部接口

### StoreRetriever

能够根据key得到 KeyInformation

### IndexRetriever

能够根据key 和 store得到 KeyInformation
根据store 得到 StoreRetriever

### Retriever

根据 index 得到IndexRetriever

这几个比较混乱。主要实现在 IndexInfoRetriever 中。



## IndexSerializer.IndexInfoRetriever

IndexInfoRetriever 继承自 KeyInformation.Retriever，只有一个 get 方法：

```java
public static class IndexInfoRetriever implements KeyInformation.Retriever {

    private final StandardJanusGraphTx transaction;

    private IndexInfoRetriever(StandardJanusGraphTx tx) {
        Preconditions.checkNotNull(tx);
        transaction=tx;
    }

    @Override
    public KeyInformation.IndexRetriever get(final String index) {
        return new KeyInformation.IndexRetriever() {

            final Map<String,KeyInformation.StoreRetriever> indexes = new ConcurrentHashMap<>();

            @Override
            public KeyInformation get(String store, String key) {
                return get(store).get(key);
            }

            @Override
            public KeyInformation.StoreRetriever get(final String store) {
                if (indexes.get(store)==null) {
                    Preconditions.checkState(transaction!=null,"Retriever has not been initialized");
                    final MixedIndexType extIndex = getMixedIndex(store, transaction);
                    assert extIndex.getBackingIndexName().equals(index);
                    final ImmutableMap.Builder<String,KeyInformation> b = ImmutableMap.builder();
                    for (final ParameterIndexField field : extIndex.getFieldKeys()) b.put(key2Field(field),getKeyInformation(field));
                    final ImmutableMap<String,KeyInformation> infoMap = b.build();
                    final KeyInformation.StoreRetriever storeRetriever = infoMap::get;
                    indexes.put(store,storeRetriever);
                }
                return indexes.get(store);
            }

        };
    }
}
```

通过看代码，我们发现其实都是几个map。
1. 首先 IndexInfoRetriever 里面有个 `final Map<String,KeyInformation.StoreRetriever> indexes = new ConcurrentHashMap<>();` 
2. KeyInformation.StoreRetriever 实际上也就是一个 `final ImmutableMap<String,KeyInformation> infoMap = b.build();` 
3. 调用 IndexInfoRetriever 的 get 方法，会调用getMixedIndex(store, transaction); 也就是说这个得到的只是 MixedIndexType。

## RecordEntry

这个类 有三个属性，分别是 long relationId, Object value, PropertyKey key，这应该就代表了待建索引的一个记录。

## IndexRecords

public static class IndexRecords extends ArrayList<RecordEntry[]> 

看上去像一个二维数组，记录索引的更新。

## IndexUpdate

这个类主要是提供一些计算索引更新的工具方法。


## 索引回顾

我们先回顾一下相关知识，主要是我们建索引的时候发生了什么。`JanusGraphIndex nameIndex = management.buildIndex("name", Vertex.class).addKey(name).buildCompositeIndex();`

```java
private JanusGraphIndex createCompositeIndex(String indexName, ElementCategory elementCategory, boolean unique, JanusGraphSchemaType constraint, PropertyKey... keys) {
    // 
    Preconditions.checkArgument(!unique || elementCategory == ElementCategory.VERTEX, "Unique indexes can only be created on vertices [%s]", indexName);
    boolean allSingleKeys = true;
    boolean oneNewKey = false;
    for (PropertyKey key : keys) {
        if (key.cardinality() != Cardinality.SINGLE) allSingleKeys = false;
        if (key.isNew()) oneNewKey = true;
        else updatedTypes.add((PropertyKeyVertex) key);
    }

    Cardinality indexCardinality;
    if (unique) indexCardinality = Cardinality.SINGLE;
    else indexCardinality = (allSingleKeys ? Cardinality.SET : Cardinality.LIST);

    boolean canIndexBeEnabled = oneNewKey || (constraint != null && constraint.isNew());

    TypeDefinitionMap def = new TypeDefinitionMap();
    def.setValue(TypeDefinitionCategory.INTERNAL_INDEX, true);
    def.setValue(TypeDefinitionCategory.ELEMENT_CATEGORY, elementCategory);
    def.setValue(TypeDefinitionCategory.BACKING_INDEX, Token.INTERNAL_INDEX_NAME);
    def.setValue(TypeDefinitionCategory.INDEXSTORE_NAME, indexName);
    def.setValue(TypeDefinitionCategory.INDEX_CARDINALITY, indexCardinality);
    def.setValue(TypeDefinitionCategory.STATUS, canIndexBeEnabled ? SchemaStatus.ENABLED : SchemaStatus.INSTALLED);
    // 新建一个顶点。
    JanusGraphSchemaVertex indexVertex = transaction.makeSchemaVertex(JanusGraphSchemaCategory.GRAPHINDEX, indexName, def);
    for (int i = 0; i < keys.length; i++) {
        Parameter[] paras = {ParameterType.INDEX_POSITION.getParameter(i)};
        // 添加边，顶点分别是两个 index 和 propertykey
        addSchemaEdge(indexVertex, keys[i], TypeDefinitionCategory.INDEX_FIELD, paras);
    }

    Preconditions.checkArgument(constraint == null || (elementCategory.isValidConstraint(constraint) && constraint instanceof JanusGraphSchemaVertex));
    if (constraint != null) {
        // 如果加了限制 ，在添加一条边。
        addSchemaEdge(indexVertex, (JanusGraphSchemaVertex) constraint, TypeDefinitionCategory.INDEX_SCHEMA_CONSTRAINT, null);
    }
    updateSchemaVertex(indexVertex);
    JanusGraphIndexWrapper index = new JanusGraphIndexWrapper(indexVertex.asIndexType());
    if (!oneNewKey) updateIndex(index, SchemaAction.REGISTER_INDEX);
    return index;
}
```

addSchemaEdge 方法 
```java
public JanusGraphEdge addSchemaEdge(JanusGraphVertex out, JanusGraphVertex in, TypeDefinitionCategory def, Object modifier) {
    assert def.isEdge();
    // 加一条边，边的 label 是 SchemaDefinitionEdge
    JanusGraphEdge edge = addEdge(out, in, BaseLabel.SchemaDefinitionEdge);
    TypeDefinitionDescription desc = new TypeDefinitionDescription(def, modifier);
    edge.property(BaseKey.SchemaDefinitionDesc.name(), desc);
    return edge;
}
```

可以看出其实就是新建一个 顶点，添加属性，然后添加边。

## indexSerializer.getIndexUpdates(del)

我们现在就看看 index 如何序列化。

```java
public Collection<IndexUpdate> getIndexUpdates(InternalRelation relation) {
    assert relation.isNew() || relation.isRemoved();
    final Set<IndexUpdate> updates = Sets.newHashSet();
    final IndexUpdate.Type updateType = getUpdateType(relation);
    final int ttl = updateType==IndexUpdate.Type.ADD?StandardJanusGraph.getTTL(relation):0;
    for (final RelationType type : relation.getPropertyKeysDirect()) {
        if (!(type instanceof PropertyKey)) continue;
        final PropertyKey key = (PropertyKey)type;
        for (final IndexType index : ((InternalRelationType)key).getKeyIndexes()) {
            if (!indexAppliesTo(index,relation)) continue;
            IndexUpdate update;
            if (index instanceof CompositeIndexType) {
                final CompositeIndexType iIndex= (CompositeIndexType) index;
                final RecordEntry[] record = indexMatch(relation, iIndex);
                if (record==null) continue;
                update = new IndexUpdate<>(iIndex, updateType, getIndexKey(iIndex, record), getIndexEntry(iIndex, record, relation), relation);
            } else {
                assert relation.valueOrNull(key)!=null;
                if (((MixedIndexType)index).getField(key).getStatus()== SchemaStatus.DISABLED) continue;
                update = getMixedIndexUpdate(relation, key, relation.valueOrNull(key), (MixedIndexType) index, updateType);
            }
            if (ttl>0) update.setTTL(ttl);
            updates.add(update);
        }
    }
    return updates;
}
```

### CompositeIndexType

我们先看 CompositeIndexType 部分，我们发现主要就是 indexMatch 方法 和 new IndexUpdate，得到某个 relation 相关的index：

```java
public static RecordEntry[] indexMatch(JanusGraphRelation relation, CompositeIndexType index) {
    // 得到所有的key。
    final IndexField[] fields = index.getFieldKeys();
    // 新建一个对应的数组
    final RecordEntry[] match = new RecordEntry[fields.length];
    for (int i = 0; i <fields.length; i++) {
        final IndexField f = fields[i];
        final Object value = relation.valueOrNull(f.getFieldKey());
        if (value==null) return null; //No match
        match[i] = new RecordEntry(relation.longId(),value,f.getFieldKey());
    }
    return match;
}
```

总的来说还是很简单的，得到所有的索引字段的值即可，但是假如一个索引有两个字段，我们每次更新其中一个字段，都会更新一次索引，这岂不是会很麻烦。

```java
private StaticBuffer getIndexKey(CompositeIndexType index, Object[] values) {
    final DataOutput out = serializer.getDataOutput(8*DEFAULT_OBJECT_BYTELEN + 8);
    // 写入 indexType 的 ID
    VariableLong.writePositive(out, index.getID());
    final IndexField[] fields = index.getFieldKeys();
    Preconditions.checkArgument(fields.length>0 && fields.length==values.length);
    for (int i = 0; i < fields.length; i++) {
        final IndexField f = fields[i];
        final Object value = values[i];
        Preconditions.checkNotNull(value);
        // 写入 index 的值。
        if (AttributeUtil.hasGenericDataType(f.getFieldKey())) {
            out.writeClassAndObject(value);
        } else {
            assert value.getClass().equals(f.getFieldKey().dataType()) : value.getClass() + " - " + f.getFieldKey().dataType();
            out.writeObjectNotNull(value);
        }
    }
    StaticBuffer key = out.getStaticBuffer();
    if (hashKeys) key = HashingUtil.hashPrefixKey(hashLength,key);
    return key;
}
```
可以看出 compositeindex 数据的key结构，indexId+value(所有的)。

```java
private Entry getIndexEntry(CompositeIndexType index, RecordEntry[] record, JanusGraphElement element) {
    final DataOutput out = serializer.getDataOutput(1+8+8*record.length+4*8);
    out.putByte(FIRST_INDEX_COLUMN_BYTE);
    if (index.getCardinality()!=Cardinality.SINGLE) { // 代表是 SET 或者 LIST
        // 写出 value 的 id
        VariableLong.writePositive(out,element.longId());
        if (index.getCardinality()!=Cardinality.SET) { // 如果是LIST
            // 循环写出 relationId
            for (final RecordEntry re : record) {
                VariableLong.writePositive(out,re.relationId);
            }
        }
    }
    // column 和 value 的分界点。
    final int valuePosition=out.getPosition();
    if (element instanceof JanusGraphVertex) { // 如果是顶点
        VariableLong.writePositive(out,element.longId());
    } else {
        assert element instanceof JanusGraphRelation;
        final RelationIdentifier rid = (RelationIdentifier)element.id();
        final long[] longs = rid.getLongRepresentation();
        Preconditions.checkArgument(longs.length == 3 || longs.length == 4);
        for (final long aLong : longs) VariableLong.writePositive(out, aLong);
    }
    return new StaticArrayEntry(out.getStaticBuffer(),valuePosition);
}
```

这两个都比较类似的，按照固定的格式，生成看key 和value。


### getMixedIndexUpdate

上面看的是 compositeIndex 的序列化过程，还有 MixedIndex。`return new IndexUpdate<>(index, updateType, element2String(element), new IndexEntry(key2Field(index.getField(key)), value), element);`

```java
private static String element2String(Object elementId) {
    if (elementId instanceof Long) return longID2Name((Long)elementId);
    else return ((RelationIdentifier) elementId).toString();
}
```

```java
private static String key2Field(ParameterIndexField field) {
    assert field!=null;
    return ParameterType.MAPPED_NAME.findParameter(field.getParameters(),keyID2Name(field.getFieldKey()));
}
```

这两个部分有很多其他的类，比较乱，后续可以自己整理一下，但是整体意思就是得到一个 key value 的类。。

## 反序列化查询

查询过程稍微有点复杂，一般会通过读索引。后续进行分析。
