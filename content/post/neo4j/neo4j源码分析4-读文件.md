---
date: 2018-03-22
title: "neo4j源码分析4-读文件"
author: "邓子明"
tags:
    - neo4j
    - 图数据库
categories:
    - 源码分析
comment: true
---

## 一、复习
上一篇我们已经大概看了 RecordStorageEngine ，他只是 NeoStoreDataSource 的 13个梦中的一个而已，我们还要醒来继续做剩下的12个梦。

然而我们可以先看看如何读数据，写数据的。第一是找到java类 `PhysicalLogCommandReaderV3_0_2`。我们可以看到里面有很多读文件处理的方法，主要是`neostore.transaction.db.0`这个文件，好像是日志文件。

然后我们在 NodeStore 类的构造方法打断点。可以找到整个调用的栈帧：
```java
new RecordStorageEngine()
neoStores = factory.openAllNeoStores( true );
return openNeoStores( createStoreIfNotExists, StoreType.values() );
return new NeoStores( neoStoreFileName, config, idGeneratorFactory, pageCache, logProvider,fileSystemAbstraction, recordFormats, createStoreIfNotExists, storeTypes, openOptions );
for ( StoreType type : storeTypes ) getOrCreateStore( type );
store = openStore( storeType );
Object store = type.open( this );
return neoStores.createNodeStore( getStoreName() );
return initialize( new NodeStore( storeFile, config, idGeneratorFactory, pageCache, logProvider,(DynamicArrayStore) getOrCreateStore( StoreType.NODE_LABEL ), recordFormats, openOptions ) );
new CommonAbstractStore()
```

同理，我们还可以在 RelationshipStore PropertyStore TokenStore AbstractDynamicStore 等store中打上断点，了解调用栈。所有的存储文件如下：
```java
CommonAbstractStore (org.neo4j.kernel.impl.store)
RelationshipStore (org.neo4j.kernel.impl.store)
RecordingRelationshipStore in WriteTransactionCommandOrderingTest (org.neo4j.kernel.impl.transaction.state)
MyStore in CommonAbstractStoreBehaviourTest (org.neo4j.kernel.impl.store)
MetaDataStore (org.neo4j.kernel.impl.store)
AbstractDynamicStore (org.neo4j.kernel.impl.store)
DynamicArrayStore (org.neo4j.kernel.impl.store)
SchemaStore (org.neo4j.kernel.impl.store)
DynamicStringStore (org.neo4j.kernel.impl.store)
Anonymous in newTestableDynamicStore() in AbstractDynamicStoreTest (org.neo4j.kernel.impl.store)
NodeStore (org.neo4j.kernel.impl.store)
RecordingNodeStore in WriteTransactionCommandOrderingTest (org.neo4j.kernel.impl.transaction.state)
RelationshipGroupStore (org.neo4j.kernel.impl.store)
TokenStore (org.neo4j.kernel.impl.store)
LabelTokenStore (org.neo4j.kernel.impl.store)
UnusedLabelTokenStore in LabelTokenStoreTest (org.neo4j.kernel.impl.store)
PropertyKeyTokenStore (org.neo4j.kernel.impl.store)
RelationshipTypeTokenStore (org.neo4j.kernel.impl.store)
TheStore in CommonAbstractStoreTest (org.neo4j.kernel.impl.store)
PropertyStore (org.neo4j.kernel.impl.store)
RecordingPropertyStore in WriteTransactionCommandOrderingTest (org.neo4j.kernel.impl.transaction.state)
```

对应20种存储格式：
```java
AbstractBaseRecord (org.neo4j.kernel.impl.store.record)
PropertyRecord (org.neo4j.kernel.impl.store.record)
IntRecord in CommonAbstractStoreBehaviourTest (org.neo4j.kernel.impl.store)
TheRecord in CommonAbstractStoreTest (org.neo4j.kernel.impl.store)
MyRecord in BaseHighLimitRecordFormatTest (org.neo4j.kernel.impl.store.format.highlimit)
MetaDataRecord (org.neo4j.kernel.impl.store.record)
SchemaRecord (org.neo4j.kernel.impl.store.record)
DynamicRecord (org.neo4j.kernel.impl.store.record)
IndexEntry (org.neo4j.consistency.store.synthetic)
PrimitiveRecord (org.neo4j.kernel.impl.store.record)
NodeRecord (org.neo4j.kernel.impl.store.record)
NeoStoreRecord (org.neo4j.kernel.impl.store.record)
RelationshipRecord (org.neo4j.kernel.impl.store.record)
LabelScanDocument (org.neo4j.consistency.store.synthetic)
RelationshipGroupRecord (org.neo4j.kernel.impl.store.record)
RelationshipGroupCursor (org.neo4j.kernel.impl.newapi)
TokenRecord (org.neo4j.kernel.impl.store.record)
LabelTokenRecord (org.neo4j.kernel.impl.store.record)
PropertyKeyTokenRecord (org.neo4j.kernel.impl.store.record)
RelationshipTypeTokenRecord (org.neo4j.kernel.impl.store.record)
CountsEntry (org.neo4j.consistency.store.synthetic)
```

在 StoreFactory 中可以找到对应的关系。

## 二、Id文件
打开代码 CommonAbstractStore ：
```java
/**
 * Opens the {@link IdGenerator} used by this store.
 * <p>
 * Note: This method may be called both while the store has the store file mapped in the
 * page cache, and while the store file is not mapped. Implementers must therefore
 * map their own temporary PagedFile for the store file, and do their file IO through that,
 * if they need to access the data in the store file.
 */
void openIdGenerator()
{
    idGenerator = idGeneratorFactory.open( getIdFileName(), getIdType(), () -> scanForHighId(), recordFormat.getMaxId() );
}
```

IdGenerator 的功能是分配id，每一种存储格式都有自己的id，所以在 CommonAbstractStore 中都有这个属性。idGenerator负责分配和释放id，所以它里面要有最大的id，已经已经释放的id。

最大的id可以用到下一次分配id，已经释放的也可以用于分配。进一步了解功能可以在 IdGeneratorImplTest 中调试。我们可以用二进制文件编辑器打开`neostore.nodestore.db.id`看看。
```
0000 0000 0000 0000 0b00 0000 0000 0000
0000 0000 0000 0000 0100 0000 0000 0000
0200 0000 0000 0000 0300 0000 0000 0000
0400 0000 0000 0000 0500 0000 0000 0000
06
```
这里一共有 65 bytes ，第1bytes是文件头，然后8 bytes是最大的id，这里是 `00 0000 0000 0000 0b` ，然后每8Bytes就是一个释放的id，这里是从0到6。

IdGeneratorImpl 的构造方法会有一个 IdContainer ，可以分配id，可以去 IdContainerTest 的 testNextId 调试 中查看功能。
```java
try
{
    IdGeneratorImpl.createGenerator( fs, idGeneratorFile(), 0, false );
    IdGenerator idGenerator = new IdGeneratorImpl( fs, idGeneratorFile(), 3, 1000, false, IdType.NODE, () -> 0L );
    for ( long i = 0; i < 7; i++ )
    {
        assertEquals( i, idGenerator.nextId() );
    }
    idGenerator.freeId( 1 );
    idGenerator.freeId( 3 );
    idGenerator.freeId( 5 );
    assertEquals( 7L, idGenerator.nextId() );
    idGenerator.freeId( 6 );
    closeIdGenerator( idGenerator );
    idGenerator = new IdGeneratorImpl( fs, idGeneratorFile(), 5, 1000, false, IdType.NODE, () -> 0L );
    idGenerator.freeId( 2 );
    idGenerator.freeId( 4 );
    assertEquals( 1L, idGenerator.nextId() );
    idGenerator.freeId( 1 );
    assertEquals( 3L, idGenerator.nextId() );
    idGenerator.freeId( 3 );
    assertEquals( 5L, idGenerator.nextId() );
    idGenerator.freeId( 5 );
    assertEquals( 6L, idGenerator.nextId() );
    idGenerator.freeId( 6 );
    assertEquals( 8L, idGenerator.nextId() );
    idGenerator.freeId( 8 );
    assertEquals( 9L, idGenerator.nextId() );
    idGenerator.freeId( 9 );
    closeIdGenerator( idGenerator );
    idGenerator = new IdGeneratorImpl( fs, idGeneratorFile(), 3, 1000, false, IdType.NODE, () -> 0L );
    assertEquals( 6L, idGenerator.nextId() );
    assertEquals( 8L, idGenerator.nextId() );
    assertEquals( 9L, idGenerator.nextId() );
    assertEquals( 1L, idGenerator.nextId() );
    assertEquals( 3L, idGenerator.nextId() );
    assertEquals( 5L, idGenerator.nextId() );
    assertEquals( 2L, idGenerator.nextId() );
    assertEquals( 4L, idGenerator.nextId() );
    assertEquals( 10L, idGenerator.nextId() );
    assertEquals( 11L, idGenerator.nextId() );
    closeIdGenerator( idGenerator );
}
```

## 三、文件读写API

ne4j有 专用的API ，neo4j的文件有它自己的特点，不能直接使用java的API，需要定义自己的API，在 org.neo4j.io.pagecache 下。我们需要了解一下。从package-info看起。

```java
The purpose of a page cache is to cache data from files on a storage device, and keep the most often used data in
memory where access is fast. This duplicates the most popular data from the file, into memory. Assuming that not all
data can fit in memory (even though it sometimes can), the least used data will then be pushed out of memory, when
we need data that is not already in the cache. This is called eviction, and choosing what to evict is the
responsibility of the eviction algorithm that runs inside the page cache implementation.
```
pagecache的功能是从文件或者存储设备缓存数据，将最常用的放在访问最快的内存。我们最少用的数据会不在内存，当我们需要的时候，这个过程是  eviction ，选择哪个 eviction 是算法最重要的。
```
A file must first be "mapped" into the page cache, before the page cache can cache the contents of the files. When
you no longer have an immediate use for the contents of the file, it can be "unmapped."
```
文件要被 map 到cache中才能使用。

通过 org.neo4j.io.pagecache.PageCache#map(java.io.File, int, java.nio.file.OpenOption...) 方法将得到一个 {@link org.neo4j.io.pagecache.PagedFile} 对象。

一旦一个文件被映射到页面缓存，它就不再被直接通过文件系统访问，因为页面缓存将保持内存的变化，认为它正在管理唯一权威的副本。

一个文件被map多次，返回的是同一个 PageCache，对应的 reference counter +1， 

Unmapping decrements the reference counter, discarding the PagedFile from the cache if the counter reaches zero.

If the last reference was unmapped, then all dirty pages for that file will be flushed before the file is discarded from the cache。

page 是一堆data的集合，可以是 file, or the memory allocated for the page cache。We refer to these two types of pages as "file pages" and "cache pages" respectively. 

Pages are the unit of what data is popular or not, and the unit of moving data into memory, and out to storage. 

When a cache page is holding the contents of a file page, the two are said to be "bound" to one another.

每个 PagedFile 对象都有一个 translation table，逻辑上存储了page file到cache里，类似 Maps 结构，key是pageid，value是page内容。

几个类的逻辑视图如下：
```java
*     +---------------[ PageCache ]-----------------------------------+
 *     |                                                               |
 *     |  * PageSwapperFactory{ FileSystemAbstraction }                |
 *     |  * evictionThread                                             |
 *     |  * a large collection of Page objects:                        |
 *     |                                                               |
 *     |  +---------------[ Page ]----------------------------------+  |
 *     |  |                                                         |  |
 *     |  |  * usageCounter                                         |  |
 *     |  |  * some kind of read/write lock                         |  |
 *     |  |  * a cache page sized buffer                            |  |
 *     |  |  * binding metadata{ filePageId, PageSwapper }          |  |
 *     |  |                                                         |  |
 *     |  +---------------------------------------------------------+  |
 *     |                                                               |
 *     |  * linked list of mapped PagedFile instances:                 |
 *     |                                                               |
 *     |  +--------------[ PagedFile ]------------------------------+  |
 *     |  |                                                         |  |
 *     |  |  * referenceCounter                                     |  |
 *     |  |  * PageSwapper{ StoreChannel, filePageSize }            |  |
 *     |  |  * PageCursor freelists                                 |  |
 *     |  |  * translation table:                                   |  |
 *     |  |                                                         |  |
 *     |  |  +--------------[ translation table ]----------------+  |  |
 *     |  |  |                                                   |  |  |
 *     |  |  |  A translation table is basically a map from      |  |  |
 *     |  |  |  file page ids to Page objects. It is updated     |  |  |
 *     |  |  |  concurrently by page faulters and the eviction   |  |  |
 *     |  |  |  thread.                                          |  |  |
 *     |  |  |                                                   |  |  |
 *     |  |  +---------------------------------------------------+  |  |
 *     |  +---------------------------------------------------------+  |
 *     +---------------------------------------------------------------+
 *
 *     +--------------[ PageCursor ]-----------------------------------+
 *     |                                                               |
 *     |  * currentPage: Page                                          |
 *     |  * page lock metadata                                         |
 *     |                                                               |
 *     +---------------------------------------------------------------+
 
```

这里有几个重要的类，我们需要大概了解一下用法，第一个是 PageCache ，可以查看 MuninnPageCacheTest 类的测试方法。
```java
try ( MuninnPageCache pageCache = createPageCache( fs, 2, blockCacheFlush( tracer ), cursorTracerSupplier );
         PagedFile pagedFile = pageCache.map( file( "a" ), 8 ) )
   {
       try ( PageCursor cursor = pagedFile.io( 0, PF_SHARED_READ_LOCK ) )
       {
           assertTrue( cursor.next() );
       }
       cursorTracer.reportEvents();
       assertNotNull( cursorTracer.observe( Fault.class ) );
       assertEquals( 1, cursorTracer.faults() );
       assertEquals( 1, tracer.faults() );

       long clockArm = pageCache.evictPages( 1, 1, tracer.beginPageEvictions( 1 ) );
       assertThat( clockArm, is( 1L ) );
       assertNotNull( tracer.observe( Evict.class ) );
    }
```
可以看出，第一步是创建 pageCache，第二步是 pageCache 的 map 方法得到 pagedFile，然后调用 io 方法得到 PageCursor ，然后cusor是一个迭代器。

## 四、CommonAbstractStore 格式
这个是一个存储格式的基本实现类，我们现在任何一个Store上面打断点，然后在 CommonAbstractStore 中打断点，开始调试即可。以 NodeStore 为例，在构造方法打断点，在 CommonAbstractStore 的 checkAndLoadStorage 打断点。

我们找到了调用栈：
```
neoStores = factory.openAllNeoStores( true );
return openNeoStores( createStoreIfNotExists, StoreType.values() );
new NeoStores( neoStoreFileName, config, idGeneratorFactory, pageCache, logProvider,fileSystemAbstraction, recordFormats, createStoreIfNotExists, storeTypes, openOptions );
getOrCreateStore( type );
store = openStore( storeType );
Object store = type.open( this );
return neoStores.createDynamicArrayStore( getStoreName(), IdType.NODE_LABELS, GraphDatabaseSettings.label_block_size );

```
在 checkAndLoadStorage 方法上停下来，此时的storeType是 `NODE_LABEL` ，也就是节点的Label,：
```java
// /Users/dengziming/opt/soft/neo4j-community-3.2.6/data/databases/graph.db/neostore.nodestore.db.labels
try ( PagedFile pagedFile = pageCache.map( storageFileName, pageSize, ANY_PAGE_SIZE ) ) 

extractHeaderRecord( pagedFile );
createStore( pageSize );
loadStorage( filePageSize );
```



## 四、AbstractDynamicStore 文件格式
neo4j 中对于字符串等变长值的保存策略是用一组定长的 block 来保存，block之间用单向链表链接。

例如 neostore.propertystore.db.arrays 和 neostore.propertystore.db.strings 类 AbstractDynamicStore 实现了该功能，文件结构在 DynamicRecordFormat 中有解释。
```
private static final int RECORD_HEADER_SIZE = 1/*header byte*/ + 3/*# of bytes*/ + 8/*max size of next reference*/;
```



