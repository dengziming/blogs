---
date: 2018-05-03
title: "JanusGraph官网文档"
author: "邓子明"
tags:
    - JanusGraph
    - 图数据库
categories:
    - 开发经营
comment: true
---

#  一、JanusGraph Basics 

## 1.config

## Chapter 3. Getting Started

janus 使用 gremin 的基本语法，详情考：http://tinkerpop.apache.org/docs/current/reference/

基本操作：
```java
gremlin> graph = JanusGraphFactory.open('conf/janusgraph-cassandra-es.properties')
==>standardjanusgraph[cassandrathrift:[127.0.0.1]]
gremlin> GraphOfTheGodsFactory.load(graph)
==>null
gremlin> g = graph.traversal()
==>graphtraversalsource[standardjanusgraph[cassandrathrift:[127.0.0.1]], standard]
```
这里我们首先通过工厂模式 JanusGraphFactory 的 open 方法 打开一个库，然后调用 load 方法，给这个库插入数据和索引，调用 traversal 方法得到一个遍历对象 g，后续操作都基于这个 g。

这里的配置文件可以修改，如果你不适用 es 作为 索引存储，这样就使用 GraphOfTheGodsFactory.loadWithoutMixedIndex() ，背后就不需要使用索引。

### 3.3. Global Graph Indices
图查询需要有一个点作为入口，然后就是通过相应的接口进行查询：
```java
gremlin> saturn = g.V().has('name', 'saturn').next()
==>v[256]
gremlin> g.V(saturn).valueMap()
==>[name:[saturn], age:[10000]]
gremlin> g.V(saturn).in('father').in('father').values('name')
==>hercules
```
这里先用 has 方法，得到 name 为 saturn 的一个点 ，然后得到这个点的所有属性，通过入边找到他的 孙子。稍微复杂的查询：
```java
gremlin> g.E().has('place', geoWithin(Geoshape.circle(37.97, 23.72, 50)))
==>e[a9x-co8-9hx-39s][16424-battled->4240]
==>e[9vp-co8-9hx-9ns][16424-battled->12520]
gremlin> g.E().has('place', geoWithin(Geoshape.circle(37.97, 23.72, 50))).as('source').inV().as('god2').select('source').outV().as('god1').select('god1', 'god2').by('name')
==>[god1:hercules, god2:hydra]
==>[god1:hercules, god2:nemean]
```
这里先得到 place 位置在某个圆内的边，然后 通过 as 进行临时命令，然后得到在这个边对应的 in 和 out 的顶点。

Graph indices （图索引）是 janus 索引的一种，另一种索引是 vertex-centric indices ，它用来在 janus 内部加速遍历，


#### 3.3.1. Graph Traversal Examples

可以查看 http://tinkerpop.apache.org/docs/current/reference/ 

## Chapter 5. Schema and Data Modeling

### 5.1. Defining Edge Labels
To define an edge label, call makeEdgeLabel(String) on an open graph or management transaction and provide the name of the edge label as the argument. Edge label names must be unique in the graph.

```java
mgmt = graph.openManagement()
follow = mgmt.makeEdgeLabel('follow').multiplicity(MULTI).make()
mother = mgmt.makeEdgeLabel('mother').multiplicity(MANY2ONE).make()
mgmt.commit()
```

### 5.2. Defining Property Keys

call makePropertyKey(String) on an open graph or management transaction and provide the name of the property key as the argument. 

Use dataType(Class) to define the data type of a property key. 

Use cardinality(Cardinality) to define the allowed cardinality of the values associated with the key on any given vertex.

```java
mgmt = graph.openManagement()
birthDate = mgmt.makePropertyKey('birthDate').dataType(Long.class).cardinality(Cardinality.SINGLE).make()
name = mgmt.makePropertyKey('name').dataType(String.class).cardinality(Cardinality.SET).make()
sensorReading = mgmt.makePropertyKey('sensorReading').dataType(Double.class).cardinality(Cardinality.LIST).make()
mgmt.commit()
```

### 5.3. Relation Types

Edge labels and property keys are jointly referred to as relation types. 

property keys and edge labels cannot have the same name. 

There are methods in the JanusGraph API to query for the existence or retrieve relation types which encompasses both property keys and edge labels.

```java
mgmt = graph.openManagement()
if (mgmt.containsRelationType('name'))
    name = mgmt.getPropertyKey('name')
mgmt.getRelationTypes(EdgeLabel.class)
mgmt.commit()
```

### 5.4. Defining Vertex Labels

```java
mgmt = graph.openManagement()
person = mgmt.makeVertexLabel('person').make()
mgmt.commit()
// Create a labeled vertex
person = graph.addVertex(label, 'person')
// Create an unlabeled vertex
v = graph.addVertex()
graph.tx().commit()
```

### 5.5. Automatic Schema Maker

### 5.6. Changing Schema Elements

```java
mgmt = graph.openManagement()
place = mgmt.getPropertyKey('place')
mgmt.changeName(place, 'location')
mgmt.commit()
```

Note, that schema name changes may not be immediately visible in currently running transactions and other JanusGraph graph instances in the cluster. 

## Chapter 6. Gremlin Query Language

Gremlin is a path-oriented language which succinctly expresses complex graph traversals and mutation operations.

http://docs.janusgraph.org/latest/gremlin.html

### 6.1. Introductory Traversals

A Gremlin query is a chain of operations/functions that are evaluated from left to right. A simple grandfather query is provided below over the Graph of the Gods dataset 

和sql相互转换： http://sql2gremlin.com/
```java
gremlin> g.V().has('name', 'hercules').out('father').out('father').values('name')
==>saturn
```

explain:
```
g: for the current graph traversal.
V: for all vertices in the graph
has('name', 'hercules'): filters the vertices down to those with name property "hercules" (there is only one).
out('father'): traverse outgoing father edge’s from Hercules.
out('father'): traverse outgoing father edge’s from Hercules' father’s vertex (i.e. Jupiter).
name: get the name property of the "hercules" vertex’s grandfather.
```

```java
gremlin> g
==>graphtraversalsource[janusgraph[cassandrathrift:127.0.0.1], standard]
gremlin> g.V().has('name', 'hercules')
==>v[24]
gremlin> g.V().has('name', 'hercules').out('father')
==>v[16]
gremlin> g.V().has('name', 'hercules').out('father').out('father')
==>v[20]
gremlin> g.V().has('name', 'hercules').out('father').out('father').values('name')
==>saturn
```

For a sanity check, it is usually good to look at the properties of each return, not the assigned long id.

```java
gremlin> g.V().has('name', 'hercules').values('name')
==>hercules
gremlin> g.V().has('name', 'hercules').out('father').values('name')
==>jupiter
gremlin> g.V().has('name', 'hercules').out('father').out('father').values('name')
==>saturn
```

```java
gremlin> g.V().has('name', 'hercules').repeat(out('father')).emit().values('name')
==>jupiter
==>saturn
```

```java
gremlin> hercules = g.V().has('name', 'hercules').next()
==>v[1536]
gremlin> g.V(hercules).out('father', 'mother').label()
==>god
==>human
gremlin> g.V(hercules).out('battled').label()
==>monster
==>monster
==>monster
gremlin> g.V(hercules).out('battled').valueMap()
==>{name=nemean}
==>{name=hydra}
==>{name=cerberus}
```

### 6.2. Iterating the Traversal

4steps:
```
iterate() - Zero results are expected or can be ignored.
next() - Get one result. Make sure to check hasNext() first.
next(int n) - Get the next n results. Make sure to check hasNext() first.
toList() - Get all results as a list. If there are no results, an empty list is returned.
```

```java
Traversal t = g.V().has("name", "pluto"); // Define a traversal
// Note the traversal is not executed/iterated yet
Vertex pluto = null;
if (t.hasNext()) { // Check if results are available
    pluto = g.V().has("name", "pluto").next(); // Get one result
    g.V(pluto).drop().iterate(); // Execute a traversal to drop pluto from graph
}
// Note the traversal can be cloned for reuse
Traversal tt = t.asAdmin().clone();
if (tt.hasNext()) {
    System.err.println("pluto was not dropped!");
}
List<Vertex> gods = g.V().hasLabel("god").toList(); // Find all the gods
```

## Chapter 7. JanusGraph Server

JanusGraph Server 应该就是类似hive的server，能够执行远程的Gremin语句。

## Chapter 8. ConfiguredGraphFactory

应该是一个通过配置管理多个graph的工厂类。

## Chapter 9. Indexing for Better Performance

JanusGraph supports two different kinds of indexing to speed up query processing: graph indexes and vertex-centric indexes. 

Most graph queries start the traversal from a list of vertices or edges that are identified by their properties. 
Graph indexes make these global retrieval operations efficient on large graphs. 

Vertex-centric indexes speed up the actual traversal through the graph, in particular when traversing through vertices with many incident edges.

### 9.1. Graph Index

#### 9.1.1. Composite Index

### 9.1.2. Mixed indexes - 支持更多谓词查询

Mixed indexes - 支持更多谓词查询
composite indexes -等值查询

代码： `mgmt.buildIndex('nameAndAge', Vertex.class).addKey(name).addKey(age).buildMixedIndex("search")`
这里的名字 search 必须在配置中添加： index.search.backend

查询方式：
```java
g.V().has('name', textContains('hercules')).has('age', inside(20, 50))
g.V().has('name', textContains('hercules'))
g.V().has('age', lt(50))
```
Graph indexes built against newly defined property keys, i.e. property keys that are defined in the same management transaction as the index, are immediately available. Graph indexes built against property keys that are already in use require the execution of a reindex procedure to ensure that the index contains all previously added elements. Until the reindex procedure has completed, the index will not be available. It is encouraged to define graph indexes in the same transaction as the initial schema.

新定义的 properties 对应的 Graph indexes 可以马上使用，例如和 index 在一个事务中定义的 property key，马上就能使用。
对于已经在使用的 property key，需要 reindex 操作完成才能使用，所以尽量在一个事务中完成操作。

#### 9.1.3. Ordering

`g.V().has('name', textContains('hercules')).order().by('age', decr).limit(10)`

Composite Index 不支持 order，调用 order 会很耗性能。
Mixed indexes 天生支持 order

#### 9.1.4. Label Constraint

可能只想在人的 name 上面建索引，其他的顶点并没有 name 属性，这时候最好的办法就是 indexOnly 
`mgmt.buildIndex('byNameAndLabel', Vertex.class).addKey(name).indexOnly(god).buildCompositeIndex()`

### 9.2. Vertex-centric Indexes

一个顶点的入边可能有很多，遍历这些边很耗时，通过  Vertex-centric Indexes 可以得到哪些需要被选择的。

查找和 hercules battled 时间为 10-20 的人。

```java
h = g.V().has('name', 'hercules').next()
g.V(h).outE('battled').has('time', inside(10, 20)).inV()
```
这样会遍历，我们可以添加索引。我们可以： Building a vertex-centric index by time speeds up such traversal queries.

```java
graph.tx().rollback()  //Never create new indexes while a transaction is active
mgmt = graph.openManagement()
time = mgmt.getPropertyKey('time')
battled = mgmt.getEdgeLabel('battled')
mgmt.buildEdgeIndex(battled, 'battlesByTime', Direction.BOTH, Order.decr, time)
mgmt.commit()
//Wait for the index to become available
mgmt.awaitGraphIndexStatus(graph, 'battlesByTime').call()
//Reindex the existing data
mgmt = graph.openManagement()
mgmt.updateIndex(mgmt.getGraphIndex("battlesByTime"), SchemaAction.REINDEX).get()
mgmt.commit()
```

#### 9.2.1. Ordered Traversals

```java
h = g..V().has('name', 'hercules').next()
g.V(h).local(outE('battled').order().by('time', decr).limit(10)).inV().values('name')
g.V(h).local(outE('battled').has('rating', 5.0).order().by('time', decr).limit(10)).values('place')
```


## Chapter 10. Transactions

所有的操作都是在一个 transaction 里面的， graph.tx().createThreadedTx() 创建 ThreadLocal 的 transatlantion 。但并不是 ACID ，因为底层的不支持，手动模拟 ACID 也很负责。

### 10.1. Transaction Handling

```java
graph = JanusGraphFactory.open("berkeleyje:/tmp/janusgraph")
juno = graph.addVertex() //Automatically opens a new transaction
juno.property("name", "juno")
graph.tx().commit() //Commits transaction
```
这里的第二段代码自动打开了一个 transaction 。

### 10.2. Transactional Scope

图中的每个元素，例如边、顶点都是有 scope 的，按照 TinkerPop 的 transaction 约定，事务在第一条语句执行的时候自动创建，commit 或者 rollback 的时候会被关闭，
一旦关闭了，在事务中创建的元素都不能用了。但是，JanusGraph will automatically transition vertices and types into the new transactional scope:

```java
graph = JanusGraphFactory.open("berkeleyje:/tmp/janusgraph")
juno = graph.addVertex() //Automatically opens a new transaction
graph.tx().commit() //Ends transaction
juno.property("name", "juno") //Vertex is automatically transitioned
```
但是边就不能这样

### 10.3. Transaction Failures

### 10.4. Multi-Threaded Transactions

### 10.5. Concurrent Algorithms

### 10.6. Nested Transactions

### 10.7. Common Transaction Handling Problems

### 10.8. Transaction Configuration

## Chapter 11. JanusGraph Cache

## Chapter 12. Transaction Log



# Part III. Storage Backends

hbase

cassandra

```java
JanusGraph graph = JanusGraphFactory.build().
	set("storage.backend", "hbase").
	open();
```


# IV. Index Backends

25. Elasticsearch
26. Apache Solr
27. Apache Lucene

# V. Advanced Topics

# VI. JanusGraph Internals

http://docs.janusgraph.org/latest/data-model.html

## Chapter 38. JanusGraph Data Model



