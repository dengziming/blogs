---
date: 2018-07-09
title: "janusgraph源码分析7-关系存储"
author: "邓子明"
tags:
    - janusgraph
    - 源码
categories:
    - janusgraph源码分析
comment: true
---

# 基础类

## InternalRelation 和 InternalRelationType

有关类型体系很复杂，可以使用 IDEA 的显示继承体系功能，查看类图。类图比较大，不太好看。大概描述一下：
主要是 JanusGraphElement 作为顶级类，接下来还有一个 InternalElement 作为顶级的 Internal 类。
JanusGraphElement 继承的类主要分为 JanusGraphRelation,JanusGraphVertex 两个分支，前者又分为 JanusGraphVertexProperty, JanusGraphEdge 。
InternalElement 的继承类主要分为 InternalRelation ,InternalVertex 两个分支，前者又分为 JanusGraphVertexProperty, JanusGraphEdge 。
其中 Internal 开头类总是有一个 JanusGraph 开头的类作为父类。例如 InternalRelation 继承自 JanusGraphRelation。

JanusGraphVertex 比较特殊，他除了有 InternalVertex 子类以外，还有 VertexLabel 和 RelationType 两个子类。
同理 InternalVertex 的继承体系下，除了真正的实体以外，还有一个 JanusGraphSchemaVertex ，他有 VertexLabelVertex, RelationTypeVertex 两个子类，
RelationTypeVertex 又有 EdgeLabelVertex 和 PropertyKeyVertex 两个子类。还有 BaseLabel BaseKey BaseVerteLabel 等子类。

这里就需要提到我们之前说的，janus 的 schema 也是以顶点的形式保存的，顶级类就是 JanusGraphSchemaVertex ，有 VertexLabelVertex, EdgeLabelVertex 和 PropertyKeyVertex 三个实现。
他们分别代表了 VertexLabel EdgeLabel PropertyKey 的 Vertex，同时我们想想，这些 Vertex 也是 janus 的元素 也是有属性的，我们岂不是还要新建三个类，保存他们的 Property Label 等？
然后他们的 Label 也是有属性的，这样下去就子子孙孙无穷尽也。所以才有了上面的 BaseLabel BaseKey BaseVerteLabel 作为终极的 Vertex。

然后我们看一下 InternalRelation 和 InternalRelationType 的关系，InternalRelation 代表的就是一种关系，有 JanusGraphEdge 和 JanusGraphVertexProperty 两种，

例如一个用户的性别是女，也就是给一个顶点添加一个性别 `女` 的属性：
首先有两个顶点, a: InternalVertex (JanusGraphVertex)， 性别则是一个 b: PropertyKey (InternalRelationType) 也是一个 Vertex，
而 `女` 则是 property 的值，实际上就是在这两个不同类型的 Vertex 之间建立一条连接。再加上一个 value 这三个组合在一起就是一个 JanusGraphVertexProperty 。

再例如我们要给一个顶点的 VertexLabel 是 User：
首先有一个用户顶点，a: InternalVertex (JanusGraphVertex)，然后 User 也是一个建好的 schema，也就是顶点： VertexLabelVertex 。然后给他们之间建立一条关系，这个关系也是一个顶点 BaseLabel.VertexLabelEdge。

在比如给两个用户之间添加一个 Friend 的关系。
首先有两个顶点就是用户，然后新建一个 StandardEdge，然后 这两个顶点分别和这个 StandardEdge 建立一个 EdgeLabel 为 Friend 的关系。

到这里我们大概明白，其实添加 Property 就是和 和一个 PropertyKey 建立一条边，添加 Edge 就是和一个 vertex 建立一条边，添加 VertexLabel 就是和一个 VertexLabel 建立一条边。

InternalRelation 就是添加的边，可以序列化存储起来，也可以读出来反序列化成 InternalRelation。 InternalRelationType 就是类型，类型也是一个顶点， 而 PropertyKey 这种类型对应的属性都是 Base开头的。


##

## RelationCache

## StaticArrayEntry

类似 java.nio 的 ByteBuffer。

# EdgeSerializer

## writeRelation

EdgeSerializer 类主要用来写 edgestore 库，这个库序列化方式相对比较简单，但代码还是比较多。

从代码调用开始看：

```java
for (Long vertexId : mutations.keySet()) {
       Preconditions.checkArgument(vertexId > 0, "Vertex has no id: %s", vertexId);
       final List<InternalRelation> edges = mutations.get(vertexId);
       final List<Entry> additions = new ArrayList<>(edges.size());
       final List<Entry> deletions = new ArrayList<>(Math.max(10, edges.size() / 10));
       for (final InternalRelation edge : edges) {
           final InternalRelationType baseType = (InternalRelationType) edge.getType();
           assert baseType.getBaseType()==null;

           for (InternalRelationType type : baseType.getRelationIndexes()) {
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

这是java类 StandardJanusGraph 写数据 的代码。可以看出写数据之前是需要调用 StaticArrayEntry entry = edgeSerializer.writeRelation(edge, type, pos, tx); 
所以接下来我么的任务就是看看这个方法，我们先看看这几个参数的意义：

InternalRelation relation, 代表一条关系，可以是 edge，也可以是 Property。
如果是edge，edge的两个顶点都会保存这条边，如果是 Property，只会有节点保存，PropertyKey 不会保存。

InternalRelationType type,  可以是 Property 和 Edge

int position, 通过调用部分代码，可以看出表示顶点在这个关系中的位置。例如 v1 -[e1]-> v2, 对于e1来讲，v1的pos是0，v2的pos是1。
TypeInspector tx 用来检测类型.

```java

public StaticArrayEntry writeRelation(InternalRelation relation, 
										InternalRelationType type, 
										int position,
										TypeInspector tx) 
										
{
    // 判断类型
    assert type==relation.getType() || (type.getBaseType() != null
            && type.getBaseType().equals(relation.getType()));
    // 得到方向，可以是 只有 OUT 和 IN 两种结果
    Direction dir = EdgeDirection.fromPosition(position);
    
    // isUnidirected 方法是判断是不是这个方向的。
    Preconditions.checkArgument(type.isUnidirected(Direction.BOTH) || type.isUnidirected(dir));
    
    // 得到 type 的id，注意 JanusGraph 中的schema 也是以顶点的形式存储，也有 id。
    long typeId = type.longId();
    
    // 得到 PROPERTY_DIR 或者 EDGE_OUT_DIR 或者 EDGE_IN_DIR
    DirectionID dirID = getDirID(dir, relation.isProperty() ? RelationCategory.PROPERTY : RelationCategory.EDGE);

    // 得到一个输出，实际就是 byte 数组
    DataOutput out = serializer.getDataOutput(DEFAULT_CAPACITY);
    
    // 保存 key 和 value 的临界点
    int valuePosition;
    
    // 这里调用方法写入 typeId dirID isInvisibleType ，详细内容我们后面看 TODO
    IDHandler.writeRelationType(out, typeId, dirID, type.isInvisibleType());
    
    // multiplicity 代表多元性
    Multiplicity multiplicity = type.multiplicity();

    long[] sortKey = type.getSortKey();
    // 多对多关系不允许有排序的key
    assert !multiplicity.isConstrained() || sortKey.length==0: type.name();
    
    int keyStartPos = out.getPosition();
    if (!multiplicity.isConstrained()) { // isConstrained 代表是否有限制。SINGLE 和 SET 有限制，LIST 无限制。
        // 写排序key ，这个方法后面讨论 TODO
        writeInlineTypes(sortKey, relation, out, tx, InlineType.KEY);
    }
    int keyEndPos = out.getPosition();

    long relationId = relation.longId();

    //How multiplicity is handled for edges and properties is slightly different
    if (relation.isEdge()) {  // 如果是边关系
        // 得到另一个顶点的id
        long otherVertexId = relation.getVertex((position + 1) % 2).longId();
        if (multiplicity.isConstrained()) { // 非多对多
            if (multiplicity.isUnique(dir)) { // 只有一个这种类型的边。例如每个 Person只有一个父亲节点。
                valuePosition = out.getPosition(); // 得到 position
                // 写出另一个顶点的id
                VariableLong.writePositive(out, otherVertexId);
            } else { // 可能有多个关系，例如一个Person可能有多个儿子节点，再或者 SIMPLE。
                
                // 这时候从后往前写，这个方法后面讨论  TODO 
                VariableLong.writePositiveBackward(out, otherVertexId);
                valuePosition = out.getPosition();
            }
            // 然后写出 关系的 relationId
            VariableLong.writePositive(out, relationId);
        } else {// 多对多，我们的数据绝大部分都是这种情况
            //从后往前写 vertex 和 relationId。得到position
            VariableLong.writePositiveBackward(out, otherVertexId);
            VariableLong.writePositiveBackward(out, relationId);
            valuePosition = out.getPosition();
        }

/**
总结上面
SortKey是一种特殊的属性，JanusGraph允许在定义Edge Label时指定其中的一个或多个属性为Sort Key。
对于边的Sort Key属性，JanusGraph在存储时会将其存储在Relation Type ID的后面,其他所有字段的前面。
通过这种方式，可以保证一个节点的多条同一个类型的边，会按Sort Key属性排序存储。这对于一个节点有大量边时，对查询性能提升有帮助。

MULTIPLICITY为MULTI时的存储结构： 从后往前写 otherVertexId 和 relationId。其余放在 value 里面

MULTIPLICITY非MULTI且此方向存在多条边时的存储结构：从后往前写 otherVertexId，relationId 和其余放在 value里面

MULTIPLICITY非MULTI且此方向仅有一条边时的存储结构： 不记录relationId，otherVertexId 放在value 里面

我一直在思考这么设计的原因，现在想想明白了，MULTIPLICITY 为 非MULTI 时候，也就是有限制，无论是什么限制，总之两个顶点之间只能有一条该类型的边。
所以 MULTIPLICITY 为 非MULTI 的时候，将 relationId 放在 value 中。这样哪怕你重复新建这条边，只是value 变了，key并没有变，而key是存在 bigtable 的 column 中的， bigtable 有个特性就是可以直接覆盖。
MULTIPLICITY 为 MULTI 的时候，relationId 放在 key 中，每次添加新的边，column 的值不一样，这样就不会覆盖原有的边。

*/

    } else { // 如果是属性关系，得到属性的 key 和 value
        assert relation.isProperty();
        Preconditions.checkArgument(relation.isProperty());
        Object value = ((JanusGraphVertexProperty) relation).value();
        Preconditions.checkNotNull(value);
        PropertyKey key = (PropertyKey) type;
        assert key.dataType().isInstance(value);

        // 没有限制，不是 LIST 类型
        if (multiplicity.isConstrained()) {
            if (multiplicity.isUnique(dir)) { //Cardinality=SINGLE
                // property 放在 value 中
                valuePosition = out.getPosition();
                writePropertyValue(out,key,value);
            } else { //Cardinality=SET
                // property 放在 key 中
                writePropertyValue(out,key,value);
                valuePosition = out.getPosition();
            }
            // 写出 relationId
            VariableLong.writePositive(out, relationId);
        } else {
            assert multiplicity.getCardinality()== Cardinality.LIST;
            // 在key中反向写出 relationId, property 放在 value 中
            VariableLong.writePositiveBackward(out, relationId);
            valuePosition = out.getPosition();
            writePropertyValue(out,key,value);
        }
    }

/** 总结上面

Cardinality为SINGLE时的存储结构

列名只存储Property Key的ID及方向。具体的Property Value值以及Property ID(relationId)，都存放在Cell的Value中。
另外，如果该Property还有额外的 Remaining properties，也会放在Value中。Remaining properties一般不使用，仅在一些特殊场景下，用于为该Property记录更多的附加信息(比如存储元数据Edge Labe的定义等)。

PropertyKeyID 及方向整个结构的详细结构在后文中描述；占用一个或多个字段，具体格式在后文描述;及采用相同的格式，具体格式在后文描述。

Candinality为LIST时的存储结构

各个部分与Cardinality为SINGLE时的结构相似，区别在于属性的ID被放在了列名中，而不是放在Value中。

Candinality为SET存储结构

各个部分与Cardinality为SINGLE时的结构相似，区别在于属性的值被放在了列名中，而不是放在Value中。

*** /


    //Write signature 
    // 得到 relationType 所有的 signature 的 PropertyKeyid，写到 value 中
    long[] signature = type.getSignature();
    writeInlineTypes(signature, relation, out, tx, InlineType.SIGNATURE);

    //Write remaining properties
    // sortKey 和 signature 是已经写过，所以排除掉
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
        // 剩下的 value 写到值部分。
        PropertyKey t = tx.getExistingPropertyKey(tid);
        writeInline(out, t, relation.getValueDirect(t), InlineType.NORMAL);
    }
    assert valuePosition>0;

    // 返回，返回的时候需要注意根据 type.getSortOrder() 的结果进行判断，如果 DESC 需要将 key 部分反过来
    return new StaticArrayEntry(type.getSortOrder() == Order.DESC ?
                                out.getStaticBufferFlipBytes(keyStartPos, keyEndPos) :
                                out.getStaticBuffer(), valuePosition);
}
```

到这里我们基本了解了数据的存储结构，但是细节还是没了解。比如key具体多少位，每一位是啥。接下来我们需要稍微了解一下每次写的时候对应方法的细节。
我们只需要找有变量 out 的代码部分。

第一次是 `DataOutput out = serializer.getDataOutput(DEFAULT_CAPACITY)`, 这个就是创建新的Buffer，然后是 IDHandler 写部分。

```
/**
 * The edge type is written as follows: [ Invisible &amp; System (2 bit) | Relation-Type-ID (1 bit) | Relation-Type-Count (variable) | Direction-ID (1 bit)]
 * Would only need 1 bit to store relation-type-id, but using two so we can upper bound.
 * 
 * 注释说明，edge格式： Invisible & System  2bit, Relation-Type-ID 1 bit, Relation-Type-Count 变化的,Direction-ID 一位。
 * 这里有个小疑问，Relation-Type-ID 也是一个 long 类型，1bit 应该没法表示。我们在代码中看
 *
 * @param out
 * @param relationTypeId
 * @param dirID
 */
public static void writeRelationType(WriteBuffer out, long relationTypeId, DirectionID dirID, boolean invisible) {
    
    // 断言判断
    assert relationTypeId > 0 && (relationTypeId << 1) > 0; //Check positive and no-overflow

    // 去掉 relationTypeId 的 padding，在后面补一位 dirID.getDirectionInt。
    long strippedId = (IDManager.stripEntireRelationTypePadding(relationTypeId) << 1) + dirID.getDirectionInt();
    {
    // 这个方法就是将 id 的 Padding 部分 去掉。
    public static long stripEntireRelationTypePadding(long id) {
        Preconditions.checkArgument(isProperRelationType(id));
        return VertexIDType.UserEdgeLabel.removePadding(id);
        {
            VertexIDType.UserEdgeLabel.removePadding(id){
                id >>> offset();// 这个 offset() 代表 id 的padding 长度，NormalVertex 是 3，EdgeLabel 是5，UserEdgeLabel 是6
            }
        }
    }
    }
    
    //
    VariableLong.writePositiveWithPrefix(out, strippedId, dirID.getPrefix(invisible, IDManager.isSystemRelationTypeId(relationTypeId)), PREFIX_BIT_LEN);
    {
    // IDManager.isSystemRelationTypeId(relationTypeId)) 判断是否是系统关系
    // getPrefix 方法如下，其实就是得到了 上面所说的数据，
    	private int getPrefix(boolean invisible, boolean systemType) {
    	    assert !systemType || invisible; // systemType implies invisible
    	    return ((systemType?0:invisible?2:1)<<1) + getRelationType();
    	}
    
    // 整个方法就是写下 prefix strippedId 。
    }
}
```

整个方法大概就清楚了，然后是 writeInlineTypes 和 writeInline ，writePropertyValue ，和上面的方法类似。
然后是 VariableLong.writePositiveBackward(out, otherVertexId); 和 VariableLong.writePositive(out, otherVertexId);

```java
public static void writePositive(WriteBuffer out, final long value) {
    assert value >= 0;
    writeUnsigned(out, value);
    
    	/** writeUnsigned 方法 */
    	{
    	private static void writeUnsigned(WriteBuffer out, final long value) {
    	    writeUnsigned(out, unsignedBlockBitLength(value), value);
    	    
    	    /** unsignedBlockBitLength 最终是 block 的数量 * 7 */
    	    {
    	    return unsignedNumBlocks(value)*7;
    	    	/** unsignedNumBlocks 求 block 数量 */
    	    	{
    	    	     return numVariableBlocks(unsignedBitLength(value));
    	    	     {
    	    	     // 得到数据去掉所有0 的位数，如果是0有1位。也就是无符号位数
    	    	     unsignedBitLength(value){
    	    	         return (value == 0) ? 1 : Long.SIZE - Long.numberOfLeadingZeros(value);
    	    	     }
    	    	     /** 这个方法返回 位数－1 除以 7 再加一，
    	    	     简单理解 ,就是第一个bit一个 block，剩下每7bit 一个 block
    	    	     实际上是除以七进一。
    	    	      */
    	    	     numVariableBlocks{
    	    	         return (numBits - 1) / 7 + 1;
    	    	     }
    	    	     }
    	    	}
    	    }
    	    
    	    /** writeUnsigned 方法 */
    	    {
    		private static void writeUnsigned(WriteBuffer out, int offset, final long value) {
        		assert offset % 7 == 0;
        		while (offset > 0) { // offset 就是上面求的 block 数量 * 7
        		    offset -= 7; // 一次写 7 位。
        		    
        		    byte b = (byte) ((value >>> offset) & BIT_MASK); // 左移 offset 与 01111111 进行 `逻辑与` 操作。
        		    if (offset == 0) {
        		        b = (byte) (b | STOP_MASK);  // 如果是最后一位，再与 -128(111111...1110000000) 进行或操作,这个操作的结果会得到一个类似补码的数据
        		    }
        		    out.putByte(b);
    		    }
    		}
    	    }
    	}
    }
}
/**
综上所述, 整个写 long 的方法，首先是计算数据的 block 数，每 7 位一个block。
写出的时候，每次写一个 byte(8bit)，其中一个block 7bit，再加一个占位符(0)。最后再与 -128(111111...1110000000) 进行或操作，
例如 72 会变成 -56， 满足 72 - (-56) = 128，这应该是补码还是反码记不清了。
*/

```

然后是  VariableLong.writePositiveBackward(out, otherVertexId)

```java

/**
 * The format used is this:
 * - The first bit indicates whether this is the first block (reading backwards, this would be the stop criterion)
 * - In the first byte, the 3 bits after the first bit indicate the number of bytes written minus 3 (since 3 is
 * the minimum number of bytes written. So, if the 3 bits are 010 = 2 => 5 bytes written. The value is aligned to
 * the left to ensure that this encoding is byte order preserving.
 *
 *  根据注释，第一 bit 表示是否是第一个 block （往后读需要一个停止标识），紧接着代表数据的位数。
 * 
 * @param out
 * @param value
 */
private static void writeUnsignedBackward(WriteBuffer out, final long value) {
    
    int numBytes = unsignedBackwardLength(value);
    /** unsignedBackwardLength 这个类似上面，得到最少的 bytes 数量。可以看出至少有3 bytes。*/
    {
        int bitLength = unsignedBitLength(value); // 这个上面已经看过。
        assert bitLength > 0 && bitLength <= 64;
        return Math.max(3, 1 + (bitLength <= 4 ? 0 : (1 + (bitLength - 5) / 7)));
    }
    int prefixLen = numBytes - 3;
    assert prefixLen >= 0 && prefixLen < 8; //Consumes 3 bits
    //Prepare first byte
    byte b = (byte)((prefixLen << 4) | 0x80); //stop marker (first bit) and length
    for (int i = numBytes - 1; i >= 0; i--) {
        b = (byte)(b | (0x7F & (value >>> (i * 7)))); // 左移 i*7 位，和 0x7F 进行逻辑与，实际上就是取七位。
        out.putByte(b);
        b = 0;
    }
}

```

上面的对比我们看出 writePositiveBackward 和 writePositive 的差别在于 writePositiveBackward 把停止标识放在了开头，writePositive 放在结尾。

看完序列化的代码我们可以大概知道存储的格式，我们整理一下。序列化的步骤在 writeRelation 中，首先写出Relation 的方向、可见性、schemaId，然后如果有sortKey写出sortKey的值，
然后判断是Edge 还是Property，根据他们的 multiplicity 处理有所不同。详情上面已经有了。最后还要写出 signature 和剩下的属性。


## readRelation

和 writeRelation 对应的是 readRelation，相关调用如下，主要是 readRelation 方法。

```java
/**
 * Returns the list of adjacent vertex ids for this query. By reading those ids
 * from the entries directly (without creating objects) we get much better performance.
 *
 * @return
 */
public VertexList vertexIds() {
    LongArrayList list = new LongArrayList();
    long previousId = 0;
    for (Long id : Iterables.transform(this,new Function<Entry, Long>() {
        @Nullable
        @Override
        public Long apply(@Nullable Entry entry) {
            return edgeSerializer.readRelation(entry,true,tx).getOtherVertexId();
        }
    })) {
        list.add(id);
        if (id>=previousId && previousId>=0) previousId=id;
        else previousId=-1;
    }
    return new VertexLongList(tx,list,previousId>=0);
}
```

进入方法发现核心就是一个 parseRelation 方法：

```java

@Override
public RelationCache parseRelation(Entry data, boolean excludeProperties, TypeInspector tx) {
    ReadBuffer in = data.asReadBuffer();

    LongObjectHashMap properties = excludeProperties ? null : new LongObjectHashMap(4);
    
    // 第一步，读取关系类型。就是上面的写进去的 三位prefix+typeId ，包括方向，类型，可见性
    RelationTypeParse typeAndDir = IDHandler.readRelationType(in);

    long typeId = typeAndDir.typeId;
    Direction dir = typeAndDir.dirID.getDirection();

   // 根据id 查询对应的类型
    RelationType relationType = tx.getExistingRelationType(typeId);
    InternalRelationType def = (InternalRelationType) relationType;
    Multiplicity multiplicity = def.multiplicity();
    long[] keySignature = def.getSortKey();

    long relationId;
    Object other;
    int startKeyPos = in.getPosition();
    int endKeyPos = 0;
    
    // 这里和前面写的对应， 分别读取
    if (relationType.isEdgeLabel()) {
        long otherVertexId;
        if (multiplicity.isConstrained()) {
            if (multiplicity.isUnique(dir)) { 
                otherVertexId = VariableLong.readPositive(in);
            } else {
                in.movePositionTo(data.getValuePosition());
                otherVertexId = VariableLong.readPositiveBackward(in);
                in.movePositionTo(data.getValuePosition());
            }
            relationId = VariableLong.readPositive(in);
        } else {
            in.movePositionTo(data.getValuePosition());

            relationId = VariableLong.readPositiveBackward(in);
            otherVertexId = VariableLong.readPositiveBackward(in);
            endKeyPos = in.getPosition();
            in.movePositionTo(data.getValuePosition());
        }
        other = otherVertexId;
    } else {
        assert relationType.isPropertyKey();
        PropertyKey key = (PropertyKey) relationType;

        if (multiplicity.isConstrained()) {
            other = readPropertyValue(in,key);
            relationId = VariableLong.readPositive(in);
        } else {
            in.movePositionTo(data.getValuePosition());
            relationId = VariableLong.readPositiveBackward(in);
            endKeyPos = in.getPosition();
            in.movePositionTo(data.getValuePosition());
            other = readPropertyValue(in,key);
        }
        Preconditions.checkState(other!=null,
            "Encountered error in deserializer [null value returned]. Check serializer compatibility.");
    }
    assert other!=null;

    // 
    if (!excludeProperties && !multiplicity.isConstrained() && keySignature.length>0) {
        int currentPos = in.getPosition();
        //Read sort key which only exists if type is not unique in this direction
        assert endKeyPos>startKeyPos;
        int keyLength = endKeyPos-startKeyPos; //after reading the ids, we are on the last byte of the key
        in.movePositionTo(startKeyPos);
        ReadBuffer inKey = in;
        if (def.getSortOrder()== Order.DESC) inKey = in.subrange(keyLength,true);
        readInlineTypes(keySignature, properties, inKey, tx, InlineType.KEY);
        in.movePositionTo(currentPos);
    }

    if (!excludeProperties) {
        //read value signature
        readInlineTypes(def.getSignature(), properties, in, tx, InlineType.SIGNATURE);

        //Third: read rest
        while (in.hasRemaining()) {
            PropertyKey type = tx.getExistingPropertyKey(IDHandler.readInlineRelationType(in));
            Object propertyValue = readInline(in, type, InlineType.NORMAL);
            assert propertyValue != null;
            properties.put(type.longId(), propertyValue);
        }

        if (data.hasMetaData()) {
            for (Map.Entry<EntryMetaData,Object> metas : data.getMetaData().entrySet()) {
                ImplicitKey key = ImplicitKey.MetaData2ImplicitKey.get(metas.getKey());
                if (key!=null) {
                    assert metas.getValue()!=null;
                    properties.put(key.longId(),metas.getValue());
                }
            }
        }
    }

    return new RelationCache(dir, typeId, relationId, other, properties);
}

```

我们可以看出，如果你熟悉上面的readRelation，就是 反过来读一遍。