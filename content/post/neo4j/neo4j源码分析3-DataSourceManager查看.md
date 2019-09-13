---
date: 2018-03-22
title: "neo4j源码分析3-LifeCycle查看"
author: "邓子明"
tags:
    - neo4j
    - 图数据库
categories:
    - 源码分析
comment: true
---

## 一、复习
上一篇我们说到，接下来我们就是一个一个分析 Lifecycle 的init和start方法，首先是 `GraphDatabaseFacadeFactory.initFacade`

1. `PlatformModule platform = createPlatform( storeDir, config, dependencies, graphDatabaseFacade );`

2. `EditionModule edition = editionFactory.apply( platform );`

3. `final DataSourceModule dataSource = createDataSource( platform, edition, queryEngine::get );`

4. `platform.life.start();`

这四段代码，前三段主要是新建 LifeCycle，最后一个是start是和我们整个neo4j的集群联系最紧密的。为了提高效率，我们可以将上面22个LifeCycle一个一个看。方法很简单，以第一个`0 = {LifeSupport$LifecycleInstance@3583} "org.neo4j.io.fs.FileSystemLifecycleAdapter@353d0772: STARTED"`为例，我们只需要找到新建和添加到lifeCycle的地方，打断点，然后在他的init和start方法上面打断点。

我们找到新建的调用栈，应该是 LifecycleManagingDatabase.start() -> ... -> new PlatformModule()，然后找到对应的代码：

```java
fileSystem = dependencies.satisfyDependency( createFileSystemAbstraction() );
life.add( new FileSystemLifecycleAdapter( fileSystem ) );
```

然后找到 FileSystemLifecycleAdapter 的 init和start方法，好像都是空的，然后打上断点进行调试。

另外对于我们关心的每一部分，实际上都是在这一块进行初始化和启动，一共就三个地方：PlatformModule，EditionModule，DataSourceModule。

例如我们要找neo4j的存储，可以在这三个类中寻找，我大概感觉是：PlatformModule 中新建的 dataSourceManager，在 CommunityEditionModule 中add到life中取得，

以及在 DataSourceModule 中的  new NeoStoreDataSource()  ，然后 dataSourceManager.register(NeoStoreDataSource)。仔细研究发现 dataSourceManager 也是一个LifeCycle，也有start方法，而他的instances包括了 NeoStoreDataSource，而 NeoStoreDataSource 也是一个LifeCycle，它的 instances是 start 方法中添加的。

## 二、DataSourceManager 预览

从上面的分析我们看出，一共22个LifeCycle，DataSourceManager 是最复杂的，我们就从它开始。
在 PlatformModule 的构造方法新建了 dataSourceManager。并且在后面调用 start 

### 1.准备工作

在DataSourceManager类的init和start方法打上断点，然后在 PlatformModule 的构造方法打上断点，在 CommunityEditionModule 上打断点，在 DataSourceModule打上断点。

另外我们的代码反复用到了 Dependencies 这个类，我们先大概知道一下它的方法，他有个 parent 属性，一个 resolveDependency 方法和一个 satisfyDependency 方法，

satisfyDependency方法是将一个类的所有父类放进一个map中，resolveDependency方法是调用 parent的resolveDependency，实际上是 DataSourceManager中的dependencies，这里可以暂时忽略。

### 2.开始调试

先定位到 PlatformModule 的断点 this.dataSourceManager = new DataSourceManager();新建只是初始化几个属性：
```java
    private LifeSupport life = new LifeSupport();
    private final Listeners<Listener> dsRegistrationListeners = new Listeners<>();
    private NeoStoreDataSource dataSource;
```

然后到 CommunityEditionModule 中，life.add( platformModule.dataSourceManager );将 dataSourceManager 添加到 LifeCycle 中。然后
```java
propertyKeyTokenHolder = life.add( dependencies.satisfyDependency( new DelegatingPropertyKeyTokenHolder(
        createPropertyKeyCreator( config, dataSourceManager, idGeneratorFactory ) ) ) );
labelTokenHolder = life.add( dependencies.satisfyDependency(new DelegatingLabelTokenHolder( createLabelIdCreator( config,
        dataSourceManager, idGeneratorFactory ) ) ));
relationshipTypeTokenHolder = life.add( dependencies.satisfyDependency(new DelegatingRelationshipTypeTokenHolder(
        createRelationshipTypeCreator( config, dataSourceManager, idGeneratorFactory ) ) ));
```
这几步用到了 dataSourceManager ，但是具体干啥了暂且不知道，看名字应该是属性标签和关系等存储相关，先跳过。


然后是 DataSourceModule 的 dataSourceManager.register( neoStoreDataSource );这里我们需要先看看 neoStoreDataSource 是啥。 打断点到 neoStoreDataSource = deps.satisfyDependency( new NeoStoreDataSource())，然后继续看看。

进入 NeoStoreDataSource 的构造方法，NeoStoreDataSource 也是 LifeCycle 的一个实现类，有start方法，它的构造方法好像就是做了很多赋值。

然后是 dataSourceManager.register( neoStoreDataSource )，实际上也就是赋值 this.dataSource = dataSource;

然后接下来是 ClassicCoreSPI spi = new ClassicCoreSPI( platform, dataSource, msgLog, coreAPIAvailabilityGuard );官方文档显示 ClassicCoreSPI 是 surface-layer-of-the-database

然后是 graphDatabaseFacade.init()

然后进入到了关键的 platform.life.start(); 我们知道这里的life的start方法会遍历 life 的 instances 调用init和start，其中就包括我们进行要调试的 DataSourceManager 。 

### 3.DataSourceManager的start方法。
我们已经在 DataSourceManager 中打好断点，我们已经知道他也是一个 Lifecycle ，先进入init方法：
```java
public void init() throws Throwable
    {
        life = new LifeSupport();
        life.add( dataSource ); // 这个DataSource是 NeoStoreDataSource
    }
```

然后是start方法：其实就是 life.start(),它的life里面只有 NeoStoreDataSource 一个 instance ，然后会调用它的init和start方法，然后进入 init和start，init是空的，我们在start调试。信息量比较大，做好准备。

第一步是 life = new LifeSupport();

第二步是 life.add( recoveryCleanupWorkCollector );

然后 life.add( indexConfigStore ) 和 life.add( Lifecycles.multiple( indexProviders.values() ) );

然后是 storageEngine = buildStorageEngine()， buildRecovery(), final NeoStoreKernelModule kernelModule = buildKernel(),

然后是 life.start();这里的life工有13个instance：
```java
instances = {ArrayList@5669}  size = 13
 0 = {LifeSupport$LifecycleInstance@5673} "org.neo4j.index.internal.gbptree.GroupingRecoveryCleanupWorkCollector@3b0c9195: NONE"
 1 = {LifeSupport$LifecycleInstance@5674} "org.neo4j.kernel.impl.index.IndexConfigStore@5cdd09b1: NONE"
 2 = {LifeSupport$LifecycleInstance@5675} "org.neo4j.kernel.lifecycle.Lifecycles$CombinedLifecycle@681a8b4e: NONE"
 3 = {LifeSupport$LifecycleInstance@5676} "org.neo4j.kernel.impl.storageengine.impl.recordstorage.RecordStorageEngine@305f7627: NONE"
 4 = {LifeSupport$LifecycleInstance@5677} "org.neo4j.kernel.impl.transaction.log.files.TransactionLogFiles@1bc715b8: NONE"
 5 = {LifeSupport$LifecycleInstance@5678} "org.neo4j.kernel.impl.transaction.log.BatchingTransactionAppender@24bdb479: NONE"
 6 = {LifeSupport$LifecycleInstance@5679} "org.neo4j.kernel.impl.transaction.log.checkpoint.CheckPointerImpl@7e3f95fe: NONE"
 7 = {LifeSupport$LifecycleInstance@5680} "org.neo4j.kernel.impl.transaction.log.checkpoint.CheckPointScheduler@34625ccd: NONE"
 8 = {LifeSupport$LifecycleInstance@5681} "org.neo4j.kernel.recovery.Recovery@39dcf4b0: NONE"
 9 = {LifeSupport$LifecycleInstance@5682} "org.neo4j.kernel.impl.api.KernelTransactions@21005f6c: NONE"
 10 = {LifeSupport$LifecycleInstance@5683} "org.neo4j.kernel.impl.api.KernelTransactionMonitorScheduler@32f0fba8: NONE"
 11 = {LifeSupport$LifecycleInstance@5684} "org.neo4j.kernel.impl.api.Kernel@545de5a4: NONE"
 12 = {LifeSupport$LifecycleInstance@5685} "org.neo4j.kernel.NeoStoreDataSource$2@2c1b9e4b: NONE"
```

这12个LifeCycle什么时候加进来的我们后面有时间再看吧，我们接下来又要跳进 LifeCycle 的的init和start方法，这13个 LifeCycle 先看哪一个呢？我们发现最后一个好像是他自己的内部类？我们后面再看吧。

我们先看和存储有关的 ` 3 = {LifeSupport$LifecycleInstance@5676} "org.neo4j.kernel.impl.storageengine.impl.recordstorage.RecordStorageEngine@305f7627: NONE"`

打好断点进入，这次终于没有 instance 了，感觉快进入了盗梦空间啊，直接看init：
```java
public void init() throws Throwable
{
    indexingService.init();
    labelScanStore.init();
}
```
IndexingService 的init方法，你可以选择跳进去，但是我不想跳进去了，不然进了但梦空间挑不出来。。。以后再看吧，姑且认为这个类和索引有关。

NativeLabelScanStore 的init，我也先不跳进去了。

再看start：
```java
public void start() throws Throwable
{
    neoStores.makeStoreOk();

    propertyKeyTokenHolder.setInitialTokens(
            neoStores.getPropertyKeyTokenStore().getTokens( Integer.MAX_VALUE ) );
    relationshipTypeTokenHolder.setInitialTokens(
            neoStores.getRelationshipTypeTokenStore().getTokens( Integer.MAX_VALUE ) );
    labelTokenHolder.setInitialTokens(
            neoStores.getLabelTokenStore().getTokens( Integer.MAX_VALUE ) );

    neoStores.rebuildCountStoreIfNeeded(); // TODO: move this to counts store lifecycle
    loadSchemaCache();
    indexingService.start();
    labelScanStore.start();
    idController.start();
}
```

neoStores.makeStoreOk(); 这个初始化就是读取本地存储，还是要重点查看一下：TODO

然后是：
```java
propertyKeyTokenHolder.setInitialTokens(
        neoStores.getPropertyKeyTokenStore().getTokens( Integer.MAX_VALUE ) );
relationshipTypeTokenHolder.setInitialTokens(
        neoStores.getRelationshipTypeTokenStore().getTokens( Integer.MAX_VALUE ) );
labelTokenHolder.setInitialTokens(
        neoStores.getLabelTokenStore().getTokens( Integer.MAX_VALUE ) );
```
这几段代码都可以直接用调试的估值功能直接看出具体的值。

然后是：neoStores.rebuildCountStoreIfNeeded(); 跳进去： getCounts().start();

然后是：
```java
neoStores.rebuildCountStoreIfNeeded(); // TODO: move this to counts store lifecycle
loadSchemaCache();
indexingService.start();
labelScanStore.start();
idController.start();
```
后面再细看吧。

## 三、dataSourceManager 剖析

上面我们已经看出了，AbstractNeoServer 包含两个 LifeCycle ，其中一个是 LifecycleManagingDatabase ，

LifecycleManagingDatabase  包含 22个 LifeCycle，其中一个是 dataSourceManager ， 

dataSourceManager 只包含一个 LifeCycle NeoStoreDataSource ，

NeoStoreDataSource 里面有 13 个 LifeCycle ， 其中有和存储有关的 RecordStorageEngine 。 
它的构造方法中有一句：neoStores = factory.openAllNeoStores( true ); 实际上会创建各种store，
相关的 store 和构造方法可以看：org.neo4j.kernel.impl.store.StoreType，以 NodeStore 为例，会先新建，然后 init。

RecordStorageEngine 中有和存储相关的很多属性和方法。分别在构造方法赋值，init和start方法进行初始化和启动工作。

这就是整个盗梦空间的五层梦，接下来我们只能从最深的一层反着往回查看了。

### 1. RecordStorageEngine 分析

首先它的父类是 StorageEngine ： A StorageEngine provides the functionality to durably store data, and read it back.负责持久化和读数据，里面的抽象方法注释可以好好阅读。

#### (1). StorageEngine 预览
storeReadLayer() , return an interface for accessing data previously applied to this storage. 返回读取之前放进storage的数据的接口。

allocateCommandCreationContext(), 保存需要多次执行的命令的上下文

createCommands(),返回一系列在当前的事务状态下进行改变的`StorageCommand`命令，CommandsToApply 命令可以通过调用apply方法放进存储中。

apply()，执行一系列的命令到存储，

其他的暂时忽略。

#### (2). RecordStorageEngine 属性查看
```java
private final StoreReadLayer storeLayer;
private final IndexingService indexingService;
private final NeoStores neoStores;
private final PropertyKeyTokenHolder propertyKeyTokenHolder;
private final RelationshipTypeTokenHolder relationshipTypeTokenHolder;
private final LabelTokenHolder labelTokenHolder;
private final DatabaseHealth databaseHealth;
private final IndexConfigStore indexConfigStore;
private final SchemaCache schemaCache;
private final IntegrityValidator integrityValidator;
private final CacheAccessBackDoor cacheAccess;
private final LabelScanStore labelScanStore;
private final SchemaIndexProviderMap schemaIndexProviderMap;
private final ExplicitIndexApplierLookup explicitIndexApplierLookup;
private final SchemaState schemaState;
private final SchemaStorage schemaStorage;
private final ConstraintSemantics constraintSemantics;
private final IdOrderingQueue explicitIndexTransactionOrdering;
private final LockService lockService;
private final WorkSync<Supplier<LabelScanWriter>,LabelUpdateWork> labelScanStoreSync;
private final CommandReaderFactory commandReaderFactory;
private final WorkSync<IndexingUpdateService,IndexUpdatesWork> indexUpdatesSync;
private final IndexStoreView indexStoreView;
private final ExplicitIndexProviderLookup explicitIndexProviderLookup;
private final PropertyPhysicalToLogicalConverter indexUpdatesConverter;
private final Supplier<StorageStatement> storeStatementSupplier;
private final IdController idController;
private final int denseNodeThreshold;
private final int recordIdBatchSize;
```

这些field赋值是在 NeoStoreDataSource#buildStorageEngine ：
```java
    private StorageEngine buildStorageEngine(
            PropertyKeyTokenHolder propertyKeyTokenHolder, LabelTokenHolder labelTokens,
            RelationshipTypeTokenHolder relationshipTypeTokens,
            ExplicitIndexProviderLookup explicitIndexProviderLookup, IndexConfigStore indexConfigStore,
            SchemaState schemaState, SynchronizedArrayIdOrderingQueue explicitIndexTransactionOrdering, OperationalMode operationalMode )
    {
        RecordStorageEngine storageEngine =
                new RecordStorageEngine( storeDir, config, pageCache, fs, logProvider, propertyKeyTokenHolder,
                        labelTokens, relationshipTypeTokens, schemaState, constraintSemantics, scheduler,
                        tokenNameLookup, lockService, schemaIndexProviderMap, indexingServiceMonitor, databaseHealth,
                        explicitIndexProviderLookup, indexConfigStore,
                        explicitIndexTransactionOrdering, idGeneratorFactory, idController, monitors, recoveryCleanupWorkCollector,
                        operationalMode );

        // We pretend that the storage engine abstract hides all details within it. Whereas that's mostly
        // true it's not entirely true for the time being. As long as we need this call below, which
        // makes available one or more internal things to the outside world, there are leaks to plug.
        storageEngine.satisfyDependencies( dependencies );

        return life.add( storageEngine );
    }
```
调试得到初始值：
```java
storeDir = {File@1774} "/Users/dengziming/opt/soft/neo4j-community-3.2.6/data/databases/graph.db"
config = {Config@1362} "dbms.connector.bolt.enabled=true, dbms.connector.http.enabled=true, dbms.connector.https.enabled=true, dbms.connectors.default_listen_address=localhost, dbms.directories.import=import, dbms.jvm.additional=-Dunsupported.dbms.udc.source=tarball, dbms.security.auth_enabled=true, dbms.tx_log.rotation.retention_policy=1 days, dbms.windows_service_name=neo4j, unsupported.dbms.block_size.array_properties=120, unsupported.dbms.block_size.labels=56, unsupported.dbms.block_size.strings=120, unsupported.dbms.directories.neo4j_home=/Users/dengziming/opt/soft/neo4j-community-3.2.6, unsupported.dbms.edition=community"
pageCache = {MuninnPageCache@1773} "MuninnPageCache[ \n Page[ id = 0, address = 4516331520, filePageId = 0, swapperId = 1, usageCounter = 1 ] OffHeapPageLock[Flush: 0, Excl: 0, Mod: 0, Ws: 0, S: 1]\n Page[ id = 1, address = 4516339712, filePageId = 0, swapperId = 2, usageCounter = 1 ] OffHeapPageLock[Flush: 0, Excl: 0, Mod: 0, Ws: 0, S: 1]\n Page[ id = 2, address = 4516347904, filePageId = 0, swapperId = 3, usageCounter = 1 ] OffHeapPageLock[Flush: 0, Excl: 0, Mod: 0, Ws: 0, S: 1]\n Page[ id = 3, address = 0, filePageId = -1, swapperId = 0, usageCounter = 0 ] OffHeapPageLock[Flush: 0, Excl: 1, Mod: 0, Ws: 0, S: 0]\n Page[ id = 4, address = 0, filePageId = -1, swapperId = 0, usageCounter = 0 ] OffHeapPageLock[Flush: 0, Excl: 1, Mod: 0, Ws: 0, S: 0]\n Page[ id = 5, address = 0, filePageId = -1, swapperId = 0, usageCounter = 0 ] OffHeapPageLock[Flush: 0, Excl: 1, Mod: 0, Ws: 0, S: 0]\n Page[ id = 6, address = 0, filePageId = -1, swapperId = 0, usageCounter = 0 ] OffHeapPageLock[Flush: 0, Excl: 1, Mod: 0, Ws: 0, S: 0]\n Page[ id = 7"
fs = {DefaultFileSystemAbstraction@1772} 
logProvider = {FormattedLogProvider@3221} 
propertyKeyTokenHolder = {DelegatingPropertyKeyTokenHolder@2506} 
labelTokens = {DelegatingLabelTokenHolder@2495} 
relationshipTypeTokens = {DelegatingRelationshipTypeTokenHolder@2480} 
schemaState = {DatabaseSchemaState@3478} 
constraintSemantics = {StandardConstraintSemantics@2499} 
scheduler = {Neo4jJobScheduler@1778} 
tokenNameLookup = {NonTransactionalTokenNameLookup@2487} 
lockService = {ReentrantLockService@3222} 
indexProviderMap = {DefaultSchemaIndexProviderMap@3483} 
indexingServiceMonitor = {$Proxy16@3223} "null"
databaseHealth = {DatabaseHealth@2485} 
explicitIndexProviderLookup = {NeoStoreDataSource$1@3226} 
indexConfigStore = {IndexConfigStore@3473} 
explicitIndexTransactionOrdering = {SynchronizedArrayIdOrderingQueue@3479} 
idGeneratorFactory = {BufferingIdGeneratorFactory@2494} 
idController = {BufferedIdController@2493} 
monitors = {Monitors@2498} 
recoveryCleanupWorkCollector = {GroupingRecoveryCleanupWorkCollector@2501} 
operationalMode = {OperationalMode@2489} "single"
```

然后构造方法走完了：
```java
storeDir = {File@1774} "/Users/dengziming/opt/soft/neo4j-community-3.2.6/data/databases/graph.db"
config = {Config@1362} "dbms.connector.bolt.enabled=true, dbms.connector.http.enabled=true, dbms.connector.https.enabled=true, dbms.connectors.default_listen_address=localhost, dbms.directories.import=import, dbms.jvm.additional=-Dunsupported.dbms.udc.source=tarball, dbms.security.auth_enabled=true, dbms.tx_log.rotation.retention_policy=1 days, dbms.windows_service_name=neo4j, unsupported.dbms.block_size.array_properties=120, unsupported.dbms.block_size.labels=56, unsupported.dbms.block_size.strings=120, unsupported.dbms.directories.neo4j_home=/Users/dengziming/opt/soft/neo4j-community-3.2.6, unsupported.dbms.edition=community"
pageCache = {MuninnPageCache@1773} Method threw 'java.lang.OutOfMemoryError' exception. Cannot evaluate org.neo4j.io.pagecache.impl.muninn.MuninnPageCache.toString()
fs = {DefaultFileSystemAbstraction@1772} 
logProvider = {FormattedLogProvider@3221} 
propertyKeyTokenHolder = {DelegatingPropertyKeyTokenHolder@2506} 
labelTokens = {DelegatingLabelTokenHolder@2495} 
relationshipTypeTokens = {DelegatingRelationshipTypeTokenHolder@2480} 
schemaState = {DatabaseSchemaState@3478} 
constraintSemantics = {StandardConstraintSemantics@2499} 
scheduler = {Neo4jJobScheduler@1778} 
tokenNameLookup = {NonTransactionalTokenNameLookup@2487} 
lockService = {ReentrantLockService@3222} 
indexProviderMap = {DefaultSchemaIndexProviderMap@3483} 
indexingServiceMonitor = {$Proxy16@3223} "null"
databaseHealth = {DatabaseHealth@2485} 
explicitIndexProviderLookup = {NeoStoreDataSource$1@3226} 
indexConfigStore = {IndexConfigStore@3473} 
explicitIndexTransactionOrdering = {SynchronizedArrayIdOrderingQueue@3479} 
idGeneratorFactory = {BufferingIdGeneratorFactory@2494} 
idController = {BufferedIdController@2493} 
monitors = {Monitors@2498} 
recoveryCleanupWorkCollector = {GroupingRecoveryCleanupWorkCollector@2501} 
operationalMode = {OperationalMode@2489} "single"
factory = {StoreFactory@3550} 
neoStoreIndexStoreView = {NeoStoreIndexStoreView@3785} 
readOnly = {Boolean@3786} "false"
neoStores = {NeoStores@3549} 
```

然后会调用 init 和 start 方法。

#### (3). RecordStorageEngine 属性分析

1. storeDir

File类型，一开始启动参数设置的路径

2. Config 

配置

3. pageCache = {MuninnPageCache@1773}

pageCache = {MuninnPageCache@1773} "MuninnPageCache[ \n Page[ id = 0, address = 4516331520, filePageId = 0, swapperId = 1, usageCounter = 1 ] OffHeapPageLock[Flush: 0, Excl: 0, Mod: 0, Ws: 0, S: 1]\n Page[ id = 1, address = 4516339712, filePageId = 0, swapperId = 2, usageCounter = 1 ] OffHeapPageLock[Flush: 0, Excl: 0, Mod: 0, Ws: 0, S: 1]\n Page[ id = 2, address = 4516347904, filePageId = 0, swapperId = 3, usageCounter = 1 ] OffHeapPageLock[Flush: 0, Excl: 0, Mod: 0, Ws: 0, S: 1]\n Page[ id = 3, address = 0, filePageId = -1, swapperId = 0, usageCounter = 0 ] OffHeapPageLock[Flush: 0, Excl: 1, Mod: 0, Ws: 0, S: 0]\n Page[ id = 4, address = 0, filePageId = -1, swapperId = 0, usageCounter = 0 ] OffHeapPageLock[Flush: 0, Excl: 1, Mod: 0, Ws: 0, S: 0]\n Page[ id = 5, address = 0, filePageId = -1, swapperId = 0, usageCounter = 0 ] OffHeapPageLock[Flush: 0, Excl: 1, Mod: 0, Ws: 0, S: 0]\n Page[ id = 6, address = 0, filePageId = -1, swapperId = 0, usageCounter = 0 ] OffHeapPageLock[Flush: 0, Excl: 1, Mod: 0, Ws: 0, S: 0]\n Page[ id = 7"

通过一个 re-usable cursor 来缓存和读取 cache 的内容，可以通过运行 MuninnPageCacheTest 的单元测试查看功能。

4. fs = {DefaultFileSystemAbstraction@1772} 

基于java的NIO 文件系统进行一个封装，。

5. logProvider = {FormattedLogProvider@3221} 

进行日志打印

6. TokenHolder

propertyKeyTokenHolder = {DelegatingPropertyKeyTokenHolder@2506} 
labelTokens = {DelegatingLabelTokenHolder@2495} 
relationshipTypeTokens = {DelegatingRelationshipTypeTokenHolder@2480} 
后面的 cacheAccess storeLayer 会用到这三个 。

7. schemaState = {DatabaseSchemaState@3478} 

存储一些状态，例如 cypher 的执行计划

8. constraintSemantics = {StandardConstraintSemantics@2499} 

里面的方法都是抛异常。
schemaCache 和后面的 txStateVisitor 用到了他

9. scheduler = {Neo4jJobScheduler@1778} 

里面是一个 synchronizedSet ，用于放任务。
indexingService 用到了他

10. tokenNameLookup = {NonTransactionalTokenNameLookup@2487} 

包含了上面的三个 TokenHolder
indexingService 用到了他

11. lockService = {ReentrantLockService@3222} 

一个读写锁，通过不区分读写实现同步
indexStoreView 用到了他

12. indexProviderMap = {DefaultSchemaIndexProviderMap@3483} 

提供索引
indexingService 用到了他

13. indexingServiceMonitor = {$Proxy16@3223} "null"

indexingService 用到了他

14. databaseHealth = {DatabaseHealth@2485} 

集群健康状态

15. explicitIndexProviderLookup = {NeoStoreDataSource$1@3226} 

貌似是查找索引用的。NeoStoreDataSource$1 是啥意思还没搞懂。。


16. indexConfigStore = {IndexConfigStore@3473} 

索引属性

17. explicitIndexTransactionOrdering = {SynchronizedArrayIdOrderingQueue@3479} 

和上面两个合作，

18. idGeneratorFactory = {BufferingIdGeneratorFactory@2494} 

封装 IdGenerator

StoreFactory factory = new StoreFactory( storeDir, config, idGeneratorFactory, pageCache, fs, logProvider );

19. idController = {BufferedIdController@2493} 

BufferedIdController safely free and reuse ids.

20. monitors = {Monitors@2498} 

监控

21. recoveryCleanupWorkCollector = {GroupingRecoveryCleanupWorkCollector@2501} 

略

22. operationalMode = {OperationalMode@2489} "single"

略

23. factory = {StoreFactory@3550} 

StoreFactory factory = new StoreFactory( storeDir, config, idGeneratorFactory, pageCache, fs, logProvider );

```java
    {
        this.config = config;
        this.idGeneratorFactory = idGeneratorFactory;
        this.fileSystemAbstraction = fileSystemAbstraction;
        this.recordFormats = recordFormats;
        this.openOptions = openOptions;
        new RecordFormatPropertyConfigurator( recordFormats, config ).configure();

        this.logProvider = logProvider;
        this.neoStoreFileName = new File( storeDir, storeName );
        this.pageCache = pageCache;
    }
```
存储工厂实现，也可以用来创建空工厂。

24. neoStoreIndexStoreView = {NeoStoreIndexStoreView@3785} 

25. neoStores = {NeoStores@3549} 

neoStores = factory.openAllNeoStores( true );

```java
new NeoStores( neoStoreFileName, config, idGeneratorFactory, pageCache, logProvider,
                fileSystemAbstraction, recordFormats, createStoreIfNotExists, storeTypes, openOptions );
```

#### (3). RecordStorageEngine init和start

上面我们已经大概明白了每个类的作用，首先我们：`StoreFactory factory = new StoreFactory( storeDir, config, idGeneratorFactory, pageCache, fs, logProvider );`

然后是： `neoStores = factory.openAllNeoStores( true );`

然后是从 neoStores 出发，新建一系列和存储有关的属性。
```java
indexUpdatesConverter = new PropertyPhysicalToLogicalConverter( neoStores.getPropertyStore() );
schemaStorage = new SchemaStorage( neoStores.getSchemaStore() );
NeoStoreIndexStoreView neoStoreIndexStoreView = new NeoStoreIndexStoreView( lockService, neoStores );
indexStoreView = new DynamicIndexStoreView( neoStoreIndexStoreView, labelScanStore, lockService, neoStores, logProvider );
schemaIndexProviderMap = indexProviderMap;
indexingService = IndexingServiceFactory.createIndexingService( config, scheduler, schemaIndexProviderMap,
        indexStoreView, tokenNameLookup,
        Iterators.asList( new SchemaStorage( neoStores.getSchemaStore() ).indexesGetAll() ), logProvider,
        indexingServiceMonitor, schemaState );

integrityValidator = new IntegrityValidator( neoStores, indexingService );
storeStatementSupplier = storeStatementSupplier( neoStores );
            storeLayer = new StorageLayer(
                    propertyKeyTokenHolder, labelTokens, relationshipTypeTokens,
                    schemaStorage, neoStores, indexingService,
                    storeStatementSupplier, schemaCache );
```

构造方法完了就是init和start

```java
@Override
public void init() throws Throwable
{
    indexingService.init(); -- 所以服务
    labelScanStore.init();  -- Label存储
}

@Override
public void start() throws Throwable
{
    neoStores.makeStoreOk();

    propertyKeyTokenHolder.setInitialTokens(
            neoStores.getPropertyKeyTokenStore().getTokens( Integer.MAX_VALUE ) );
    relationshipTypeTokenHolder.setInitialTokens(
            neoStores.getRelationshipTypeTokenStore().getTokens( Integer.MAX_VALUE ) );
    labelTokenHolder.setInitialTokens(
            neoStores.getLabelTokenStore().getTokens( Integer.MAX_VALUE ) );

    neoStores.rebuildCountStoreIfNeeded(); // TODO: move this to counts store lifecycle
    loadSchemaCache();
    indexingService.start();
    labelScanStore.start();
    idController.start();
}
```

一步一步看： 
1. indexingService.init();

```java
// Each index has an {@link org.neo4j.kernel.impl.store.record.IndexRule}
// 遍历每一个 IndexRule ，
IndexProxy indexProxy;

long indexId = indexRule.getId();
IndexDescriptor descriptor = indexRule.getIndexDescriptor();
SchemaIndexProvider.Descriptor providerDescriptor = indexRule.getProviderDescriptor();
SchemaIndexProvider provider = providerMap.apply( providerDescriptor );
InternalIndexState initialState = provider.getInitialState( indexId, descriptor );
indexStates.computeIfAbsent( initialState, internalIndexState -> new ArrayList<>() )
.add( new IndexLogRecord( indexId, descriptor ) );

log.debug( indexStateInfo( "init", indexId, initialState, descriptor ) );
switch ( initialState )
{
case ONLINE:
    indexProxy =
    indexProxyCreator.createOnlineIndexProxy( indexId, descriptor, providerDescriptor );
    break;
case POPULATING:
    // The database was shut down during population, or a crash has occurred, or some other sad thing.
    indexProxy = indexProxyCreator.createRecoveringIndexProxy( descriptor, providerDescriptor );
    break;
case FAILED:
    IndexPopulationFailure failure = failure( provider.getPopulationFailure( indexId ) );
    indexProxy = indexProxyCreator
            .createFailedIndexProxy( indexId, descriptor, providerDescriptor, failure );
    break;
default:
    throw new IllegalArgumentException( "" + initialState );
}
indexMap.putIndexProxy( indexId, indexProxy );
```
由于我的数据库没有建索引，所以这里就不调试了，接下来建了索引再说。

2. `labelScanStore.init();`

通过 GBPTree 实现
```java
-- which is implemented using {@link GBPTree}
@link GBPTree 是一种算法，减少树结构合并时候的冲突。
```

3. indexingService.start();
        
4. labelScanStore.start();


5. idController.start();


### 2. RecordStorageEngine 分析

上一节我们已经大概看了 RecordStorageEngine ，他只是 NeoStoreDataSource 的 13个梦中的一个而已，我们还要醒来继续做剩下的12个梦。
我看了一下 先不看了，剩下的12个重要性稍微低一点。我们先看我们最感兴趣的。

PageCache


