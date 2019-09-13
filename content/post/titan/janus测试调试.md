---
date: 2018-10-26
title: "janus官方实例调试解析"
author: "邓子明"
tags:
    - janusgraph
    - 源码
categories:
    - janusgraph
comment: true
---


## 开始


打好断点。主要类：

JanusGraphFactory.build() 建造者模式。

```java

// new GraphDatabaseConfiguration

// 创建两个 conf 对象
BasicConfiguration localBasicConfiguration = new BasicConfiguration(ROOT_NS,localConfig, BasicConfiguration.Restriction.NONE);
ModifiableConfiguration overwrite = new ModifiableConfiguration(ROOT_NS,new CommonsConfiguration(), BasicConfiguration.Restriction.NONE);

// get storeManager，根据配置反射生成。
final KeyColumnValueStoreManager storeManager = Backend.getStorageManager(localBasicConfiguration);

// conf ，需要连接数据库获得配置。
KCVSConfiguration keyColumnValueStoreConfiguration=Backend.getStandaloneGlobalConfiguration(storeManager,localBasicConfiguration);

// 后面是连接数据库进行读写和默认设置
preLoadConfiguration()

```

Builder.open() 新建 StandardJanusGraph,首先是静态代码和成员变量
```java

// TraversalStrategies 优化策略， 需要结合 tinkerpop 的代码才能理解
static {
        TraversalStrategies graphStrategies = TraversalStrategies.GlobalCache.getStrategies(Graph.class).clone()
                .addStrategies(AdjacentVertexFilterOptimizerStrategy.instance(), JanusGraphLocalQueryOptimizerStrategy.instance(), JanusGraphStepStrategy.instance());

        //Register with cache
        TraversalStrategies.GlobalCache.registerStrategies(StandardJanusGraph.class, graphStrategies);
        TraversalStrategies.GlobalCache.registerStrategies(StandardJanusGraphTx.class, graphStrategies);
    }
    
<clinit>:641, StandardJanusGraph (org.janusgraph.graphdb.database)

// Predicate 用来判断新增的 Relation 是 schema 还是 data，可以看出 BaseRelationType 是属于 schema ，而且第一个顶点是 JanusGraphSchemaVertex
private static final Predicate<InternalRelation> SCHEMA_FILTER =
        internalRelation -> internalRelation.getType() instanceof BaseRelationType && internalRelation.getVertex(0) instanceof JanusGraphSchemaVertex;
    
// NO_SCHEMA_FILTER NO_FILTER 

private final SchemaCache.StoreRetrieval typeCacheRetrieval = new SchemaCache.StoreRetrieval() {。。。}

this.backend = configuration.getBackend();

// 序列化
this.serializer = config.getSerializer();
StoreFeatures storeFeatures = backend.getStoreFeatures();
this.indexSerializer = new IndexSerializer(configuration.getConfiguration(), this.serializer,
        this.backend.getIndexInformation(), storeFeatures.isDistributed() && storeFeatures.isKeyOrdered());
this.edgeSerializer = new EdgeSerializer(this.serializer);
this.vertexExistenceQuery = edgeSerializer.getQuery(BaseKey.VertexExists, Direction.OUT, new EdgeSerializer.TypedInterval[0]).setLimit(1);
this.queryCache = new RelationQueryCache(this.edgeSerializer);
this.schemaCache = configuration.getTypeCache(typeCacheRetrieval);
 
// management 日志管理
Log managementLog = backend.getSystemMgmtLog();
// registerReader 后，就会不断的读取日志。
managementLogger = new ManagementLogger(this, managementLog, schemaCache, this.times);
managementLog.registerReader(ReadMarker.fromNow(), managementLogger);


```

Backend, 协调和配置所有后端系统

```java

private final ConcurrentHashMap<String, Locker> lockers = new ConcurrentHashMap<>();

// 并发锁创建
CONSISTENT_KEY_LOCKER_CREATOR = .....// lockerStore = storeManager.openDatabase(lockerName);// 

ASTYANAX_RECIPE_LOCKER_CREATOR = 。。

indexes = getIndexes(configuration);

managementLogManager = getKCVSLogManager(MANAGEMENT_LOG);
txLogManager = getKCVSLogManager(TRANSACTION_LOG);
userLogManager = getLogManager(USER_LOG);

KeyColumnValueStore idStore = storeManager.openDatabase(config.get(IDS_STORE_NAME));
idAuthority = new ConsistentKeyIDAuthority(idStore, storeManager, config);
KeyColumnValueStore edgeStoreRaw = storeManagerLocking.openDatabase(EDGESTORE_NAME);
KeyColumnValueStore indexStoreRaw = storeManagerLocking.openDatabase(INDEXSTORE_NAME);

txLogManager.openLog(SYSTEM_TX_LOG_NAME);

txLogStore = new NoKCVSCache(storeManager.openDatabase(SYSTEM_TX_LOG_NAME));

```

JanusGraphManagement management = graph.openManagement();

```
skip
```

