---
date: 2018-03-22
title: "neo4j存储结构分析"
author: "邓子明"
tags:
    - neo4j
    - 图数据库
categories:
    - 开发经验
comment: true
---

## 1.本文内容转自：

https://key-value-stories.blogspot.tw/2015/02/neo4j-architecture.html?view=magazine


This post compiles some information about architecture of Neo4j, the leading graph database. Research is relevant for Neo4j 2.2 version.

There are three main kinds of primitives in Neo4j: nodes, relationships and properties. Nodes are connected via relationships. Properties could be attached to both nodes and relationships. All primitives are identified by identifiers, unique among primitive kind.

Node and relationship identifiers are 35 bits in length, i. e. database could hold at most about 34 billions of nodes or relationships. Property identifiers take 36 bits (reference).

Also, relationships are typed (for example, "friend of" and "in relationship with" are two different types of relationships between "people nodes" in social network graph). Relationship types have 2-byte identifiers.

In addition, nodes could be labelled. Label is logically another one kind of entity, with own identifiers.
Main data storage
Primitives stored on disk as records. Important, that all records or primitives of a kind are equally sized. (Actually, there are more record types and dedicated stores, and they all share this property. Dynamically sized data is stored as a linked list of constantly sized records.)
Node store format
Node records are 15 bytes (* 8 = 120 bits) long:
35 bits for first relationship identifier
36 bits for first property identifier
40 bytes for label field:
If there are at most 7 labels attached to the node, and each of label identifiers takes no more than (40 - 4  = 36) / numberOfLabels, i. e. if there are 7 labels, each label id should be below 2(36 / 7) = 5 = 32, than number of labels is stored in 36..38-th bits of the label field, and the label identifiers are packed in lower 0..35-th bits.
Otherwise, if there are more than 7 labels attached to the node, or their identifiers are too big, 39-th bit of the label field, i. e. the flag, is set, and in the lower 0..35-th bits of the label field the identifier of the dynamic record with all label ids is stored.
9 bits -- some flags and reserved for future use
Relationship store format
Since node keeps only a reference to the single relationship, relationships are orginized in doubly-linked lists, that makes all relationships of some node traversable from this node. Each relationship is a part of two linked lists: a list of relationships of the first node, i. e. from which this relationship starts, and a lists of relationships of the second node, i. e. at which this relationships ends.

Relationship record take 34 bytes (* 8 = 272 bits):
35 bits of the first node identifier
35 bits of the second node identifier
35 * 4 = 140 bits of identifiers of the sibling relationships in two linked lists, this relationship participate in
16 bits of relationship type
36 bits for first property identifier
10 bits -- some flags and reserved for future use
Organazing relationships in linked lists is not particularly performant decision itself, but in some cases it becomes really disastrous -- for example, when some type of nodes has (on average) 100 relationships of type A and some relationships of type B. If we are only interested in traversal over relationships of type B, and they are occasionally clustered in the end of linked lists of the nodes of our type, we are required to traverse 100 relationships in which we are not currently interested to access the useful data.

Apparently to optimize cases like explained above, Neo4j supports another relationship layout (called dense node), in a nutshell it links relationships of each node in a tree, rather than simple linked list. In this case, "first relationship identifier" is interpreted as an identifier of a relationship group. Each relationship group is dedicated to relationships of a certain type. Relationship group record is 25 bytes (* 8 = 200 bits) long:
35 bytes of the node identifier this relationship group belongs to
16 bits of relationship type
35 bytes of the first out relationship identifier, i. e. a relationship which has the given type and starts in the node owning this relationship group
35 bytes of the first in relationship identifier, i. e. a relationship which has the given type and ends in the node owning this relationship group
35 bytes of the first loop relationship identifier.
35 bytes of the next linked relationship group of the owning node, i. e. relationship groups form a singly-linked list
1 bit for presence flag
One more byte (8 bits) apparently reserved for future use, however I'm not sure, because seems that it would be nicer to fit 24 bytes for relationship group record, because it is more "power of 2 aligned", i. e. plays better with cache lines, pages.
When relationship groups are used, relationships of any specified type and direction, could be traversed from from the node with much lesser overhead, skipping potentially a lot of relationships of the node we are not interested in during this traversal.

There is an interesting small optimization: when the node is dense, first relationship records in the doubly-linked lists, to which relationship groups point, keep the length of the doubly-linked list in place of previous relationship link, which is otherwise unused (because first relationship in the doubly-linked list point only to the next relationship in a chain).
Property store format
As nodes and relationships reference only the first their property, they are also stored as doubly-linked lists, by owning primitive entity. Property record size is 41 bytes:
36 bits of the previous linked property identifier
36 bits of the next linked property identifier
32 bytes of "payload", i. e. space where the property data itself is stored. It includes property type identifier, data encoding type and the data bytes. If the property data doesn't fit the payload (i. e. it is a long string or array), the identifier of the linked dynamic data record is placed there as well.
Neo4j supports plenty of property data formats, trying to pack the data as dense as possible, but it is not the subject of this blog post.
File buffer cache
Records of different kinds are stored in separate files. Access to disk storage is proxied with file buffer cache, or page cache:
Neo4j uses multiple file buffer caches, one for each different storage file. Each file buffer cache divides its storage file into a number of equally sized windows. Each cache window contains an even number of storage records. The cache holds the most active cache windows in memory and tracks hit vs. miss ratio for the windows. When the hit ratio of an uncached window gets higher than the miss ratio of a cached window, the cached window gets evicted and the previously uncached window is cached instead.
Quote from Caches in Neo4j document.

I would add, that the default page size is 8192 bytes, and it doesn't depend on native page size, specified for the Neo4j server process, operation system or CPU platform.

I don't understand, why Neo4j developers don't rely on OS file caches, which employ several heuristics, including LRU, to solve the same task. Probably user space implementation is more precise in decisions about page eviction, than native generic mechanism would be, and is more manageable, but, on the other hand,
Native page caching is fully transparent, i. e. relying on it simply throws a layer of complexity away from Neo4j project
Even with Java-level page cache, OS still caches the same pages underneath, i. e. work is doubled to some extent.
Summary
The approach to data storage, chosen in Neo4j has one very useful consequence: since all records are strictly of the same size, accessing records by identifiers is pretty cheap, because doesn't require any associative mapping from identifiers to record locations (hash table, tree or something else), identifiers just play as indexes in "arrays" of records.

Another strong point is impossibility of external fragmentation, records after removed primitives could always be reused.

However, there are also major disadvantages:
Database entity identifiers hardly could simultaneously be domain identifiers, unless the system was designed to use Neo4j as primary storage from the beginning.
A lot of memory overhead for storing links between records. For example, 50-80% of 34-byte relationship record is overhead (depends on how to count).
Traversing links is slow. Partially it is excused by empirical observation, that if all relationships of the node or properties of the node or relationship are stored at once, they should reside adjacent records and their traversal won't require page/cache line load on each step.
Neo4j's storage design favor reliability, versatility, agility and manageability. Apparently it is driven by initial database functional requirements and equally powerful, but more efficient approach doesn't exist (at least I don't see such). There is a basic tradeoff in systems design: more specific and constrained systems could be implemented more efficiently, than general and schema-less, like Neo4j.
Indexes
Neo4j supports indexing of nodes and relationships by labels and property values, i. e. it allows retrieving all nodes with given label and/or property value faster, than via full scan of all nodes in the database.

Apparently production implementation of indexes is fully delegated to Lucene engine. Entity identifiers are stored in Lucene documents with fields, corresponding to the indexed labels and property values. Lucene is able to search documents by individual fields and combinations of fields, empowering complex queries on Neo4j level.

I can't judge about propriety and efficiency of this solution, because I'm not familiar with Lucene implementation. This requires separate research.
Object cache
Neo4j is written in Java, known for allocation and GC issues. It has several versions of object caches, introduced to prevent too much unnecessary allocations of node, relationship and property wrapper objects. Note that Neo4j's object caches are not object pools, i. e. objects are not reused, caches only control object's lifecycle in managed memory environment (JVM).

Community strong, weak and soft cache implementations use ConcurrentHashMap with Long keys and target cached object values. weak and soft versions additionally wrap values with WeakReference or SoftReference respectively. Object eviction is left to JVM. At least one obvious optimization is possible here: specialization of ConcurrentHashMap for primitive long keys.

Enterprise hpc (high-performance cache) uses simpler data structure: basically it's just an AtomicReferenceArray of cached entities, slot for particular entity is determined as entity.id() % array.length(). On collisions, old cached objects are evicted. Eviction algorithm is also very simple: when after insertion of the new object total memory footprint of cached objects (including JVM object headers), preserved in a counter, exceeds configured limit, slots before and after current insertion index are cleared (in interleaved order), while total size of the cached objects is higher than 90% of the limit size.

It's an amusing example, how applying little knowledge about the problem and major usage patterns leads to faster solution, even with simpler implementation.
Conclusion
In my understanding, Neo4j is a reliable, agile, general-purpose database, but it is not for edge performance, despite claims on their official website. Databases that allow to specialize storage and data structures for concrete node/relationship types should be more efficient.