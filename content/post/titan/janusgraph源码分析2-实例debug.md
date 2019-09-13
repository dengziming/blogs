---
date: 2018-04-26
title: "janusgraph源码分析2-实例debug"
author: "邓子明"
tags:
    - 源码
    - janusgraph
categories:
    - 源码分析
comment: true
---

#
研究了好久的 neo4j源码，现在公司要换 janusgraph，只要半途而废开始研究 janusgraph了
`https://github.com/JanusGraph/janusgraph`和`http://janusgraph.org/`



## 一、第一遍调试

还是上次的例子 `FirstTest`：

```java
public class FirstTest {

    public static void main(String[] args) {

        /*
         * The example below will open a JanusGraph graph instance and load The Graph of the Gods dataset diagrammed above.
         * JanusGraphFactory provides a set of static open methods,
         * each of which takes a configuration as its argument and returns a graph instance.
         * This tutorial calls one of these open methods on a configuration
         * that uses the BerkeleyDB storage backend and the Elasticsearch index backend,
         * then loads The Graph of the Gods using the helper class GraphOfTheGodsFactory.
         * This section skips over the configuration details, but additional information about storage backends,
         * index backends, and their configuration are available in
         * Part III, “Storage Backends”, Part IV, “Index Backends”, and Chapter 13, Configuration Reference.
         */

        // Loading the Graph of the Gods Into JanusGraph
        JanusGraph graph = JanusGraphFactory
                .open("janusgraph-dist/src/assembly/cfilter/conf/janusgraph-berkeleyje-es.properties");

        GraphOfTheGodsFactory.load(graph);
        GraphTraversalSource g = graph.traversal();

        /*
         * The typical pattern for accessing data in a graph database is to first locate the entry point into the graph
         * using a graph index. That entry point is an element (or set of elements) 
         * — i.e. a vertex or edge. From the entry elements,
         * a Gremlin path description describes how to traverse to other elements in the graph via the explicit graph structure.
         * Given that there is a unique index on name property, the Saturn vertex can be retrieved.
         * The property map (i.e. the key/value pairs of Saturn) can then be examined.
         * As demonstrated, the Saturn vertex has a name of "saturn, " an age of 10000, and a type of "titan."
         * The grandchild of Saturn can be retrieved with a traversal that expresses:
         * "Who is Saturn’s grandchild?" (the inverse of "father" is "child"). The result is Hercules.
         */
        // Global Graph Indices
        Vertex saturn = g.V().has("name", "saturn").next();
        GraphTraversal<Vertex, Map<String, Object>> vertexMapGraphTraversal = g.V(saturn).valueMap();

        GraphTraversal<Vertex, Object> values = g.V(saturn).in("father").in("father").values("name");

        /*
         * The property place is also in a graph index. The property place is an edge property.
         * Therefore, JanusGraph can index edges in a graph index.
         * It is possible to query The Graph of the Gods for all events that have happened within 50 kilometers of Athens
          * (latitude:37.97 and long:23.72).
          * Then, given that information, which vertices were involved in those events.
         */
		System.out.println(g.E().has("place", geoWithin(Geoshape.circle(37.97, 23.72, 50))));
        System.out.println(g.E().has("place", geoWithin(Geoshape.circle(37.97, 23.72, 50)))
                .as("source").inV()
                .as("god2")
                .select("source").outV()
                .as("god1").select("god1", "god2")
                .by("name"));
    }

}
```

删除 db 文件夹，打上断点，开始debug，首先进入：JanusGraphFactory.open

JanusGraphFactory is used to open or instantiate a JanusGraph graph database.
Opens a {@link JanusGraph} database configured according to the provided configuration.

```java
public static JanusGraph open(ReadConfiguration configuration, String backupName) {
    final ModifiableConfiguration config = new ModifiableConfiguration(ROOT_NS, (WriteConfiguration) configuration, BasicConfiguration.Restriction.NONE);
    final String graphName = config.has(GRAPH_NAME) ? config.get(GRAPH_NAME) : backupName;
    final JanusGraphManager jgm = JanusGraphManagerUtility.getInstance();
    if (null != graphName) {
        Preconditions.checkState(jgm != null, JANUS_GRAPH_MANAGER_EXPECTED_STATE_MSG);
        return (JanusGraph) jgm.openGraph(graphName, gName -> new StandardJanusGraph(new GraphDatabaseConfiguration(configuration)));
    } else {
        if (jgm != null) {
            log.warn("...");
        }
        return new StandardJanusGraph(new GraphDatabaseConfiguration(configuration));
    }
}
```

前面的部分先跳过，然后进入：
```
1. return new StandardJanusGraph(new GraphDatabaseConfiguration(configuration));
    // 构造方法，分为静态代码和构造方法，这部分目前是跳过，但是后续是重点和核心。
    1. 父类：JanusGraphBlueprintsGraph
        static {
        TraversalStrategies graphStrategies = TraversalStrategies.GlobalCache.getStrategies(Graph.class).clone()
                .addStrategies(AdjacentVertexFilterOptimizerStrategy.instance(), JanusGraphLocalQueryOptimizerStrategy.instance(), JanusGraphStepStrategy.instance());

        //Register with cache
        TraversalStrategies.GlobalCache.registerStrategies(StandardJanusGraph.class, graphStrategies);
        TraversalStrategies.GlobalCache.registerStrategies(StandardJanusGraphTx.class, graphStrategies);
        }
    2. 新建配置，A graph database configuration is uniquely associated with a graph database and must not be used for multiple databases
    
    new GraphDatabaseConfiguration(configuration)
        1. storeManager 
        final KeyColumnValueStoreManager storeManager = Backend.getStorageManager(localBasicConfiguration);
        final StoreFeatures storeFeatures = storeManager.getFeatures();
        2. 检查参数，配置等
    
    3. 然后是构造方法
        1. 成员变量
        private final SchemaCache.StoreRetrieval typeCacheRetrieval = new SchemaCache.StoreRetrieval() {}
        2. backend
        this.backend = configuration.getBackend();
            1. Backend backend = new Backend(configuration);
                1. KeyColumnValueStoreManager manager = getStorageManager(configuration);
                2. indexes = getIndexes(configuration);
                
                3. //这里的 KCVS 是 keycolumnvaluestorageManager
                managementLogManager = getKCVSLogManager(MANAGEMENT_LOG);
        		txLogManager = getKCVSLogManager(TRANSACTION_LOG);
        		userLogManager = getLogManager(USER_LOG);
        		
        		4. scanner = new StandardScanner(storeManager);
                
            2. backend.initialize(configuration);
                1. store 新建
                KeyColumnValueStore idStore = storeManager.openDatabase(config.get(IDS_STORE_NAME));
                KeyColumnValueStore edgeStoreRaw = storeManagerLocking.openDatabase(EDGESTORE_NAME);
            	KeyColumnValueStore indexStoreRaw = storeManagerLocking.openDatabase(INDEXSTORE_NAME);
                
                2. cacheEnabled
                edgeStore = new NoKCVSCache(edgeStoreRaw);
                indexStore = new NoKCVSCache(indexStoreRaw);
            3. storeFeatures = backend.getStoreFeatures();
        3. 初始化
        this.idAssigner = config.getIDAssigner(backend);
        this.idManager = idAssigner.getIDManager();
        this.serializer = config.getSerializer();
        StoreFeatures storeFeatures = backend.getStoreFeatures();
        this.indexSerializer = new IndexSerializer(configuration.getConfiguration(), this.serializer,
        this.backend.getIndexInformation(), storeFeatures.isDistributed() && storeFeatures.isKeyOrdered());
        this.edgeSerializer = new EdgeSerializer(this.serializer);
        this.vertexExistenceQuery = edgeSerializer.getQuery(BaseKey.VertexExists, Direction.OUT, new EdgeSerializer.TypedInterval[0]).setLimit(1);
        this.queryCache = new RelationQueryCache(this.edgeSerializer);
        this.schemaCache = configuration.getTypeCache(typeCacheRetrieval);
        this.times = configuration.getTimestampProvider();
        
```




然后是open完成后：GraphOfTheGodsFactory.load(graph);

```java
1. 得到management
JanusGraphManagement management = graph.openManagement();
    
    1. new ManagementSystem
        1. 启动 tx
        this.transaction = (StandardJanusGraphTx) graph.buildTransaction().disableBatchLoading().start();
            1.  graph.newTransaction(immutable);
                StandardJanusGraphTx tx = new StandardJanusGraphTx(this, configuration);
            	tx.setBackendTransaction(openBackendTransaction(tx));
            	openTransactions.add(tx);
2. 得到 PropertyKey
final PropertyKey name = management.makePropertyKey("name").dataType(String.class).make();
    1. return transaction.makePropertyKey(name);
        1. return new StandardPropertyKeyMaker(this, name, indexSerializer, attributeHandler);
            1. super(tx, name, indexSerializer, attributeHandler);
    2. public StandardPropertyKeyMaker dataType(Class<?> clazz)
    3. public PropertyKey make()
        1. TypeDefinitionMap definition = makeDefinition();        
        2. return tx.makePropertyKey(getName(), definition);
            1. return (PropertyKey) makeSchemaVertex(JanusGraphSchemaCategory.PROPERTYKEY, name, definition);
                1. ... 先跳过。
            
3. 新建 index
JanusGraphManagement.IndexBuilder nameIndexBuilder = management.buildIndex("name", Vertex.class).addKey(name);
    1. 
```

调用：JanusGraphManagement management = graph.openManagement();然后：management.makeEdgeLabel("father").multiplicity(Multiplicity.MANY2ONE).make();

然后就是查询数据库：`Vertex saturn = g.V().has("name", "saturn").next();`


## 二、第2遍调试

这次我们多关注一点细节实现，包括几个部分：
```java
Backend backend = new Backend(configuration);
backend.~~~

this.idAssigner = config.getIDAssigner(backend);
this.idManager = idAssigner.getIDManager();

JanusGraphManagement management = graph.openManagement();
management.makePropertyKey("name").dataType(String.class).make();
management.buildIndex("name", Vertex.class).addKey(name);

Vertex tartarus = tx.addVertex(T.label, "location", "name", "tartarus");
jupiter.addEdge("father", saturn);


```

### Backend

```java
public StandardJanusGraph(GraphDatabaseConfiguration configuration) 
{
    this.backend = configuration.getBackend();
    {
        Backend backend = new Backend(configuration);
        {
            this.configuration = configuration;
            KeyColumnValueStoreManager manager = getStorageManager(configuration);
            {
                反射生成一个 KeyColumnValueStoreManager 实现类
            }
            indexes = getIndexes(configuration);
            {
                IndexProvider provider = getImplementationClass(config.restrictTo(index), config.get(INDEX_BACKEND,index),
                    StandardIndexProvider.getAllProviderClasses());
                -- org.janusgraph.diskstorage.es.ElasticSearchIndex
                builder.put(index, provider);
                builder.build();
            }
            storeFeatures = storeManager.getFeatures();
            {
                ...
            }
            ...
        }
        
        backend.initialize(configuration);
        {
            KeyColumnValueStore idStore = storeManager.openDatabase(config.get(IDS_STORE_NAME));
            {
                openDatabase("janusgraph_ids", EMPTY)
                {
                    if (!stores.containsKey(name) || stores.get(name).isClosed()) {
                         OrderedKeyValueStoreAdapter store = wrapKeyValueStore(manager.openDatabase(name), keyLengths);
                         {
                             public BerkeleyJEKeyValueStore openDatabase(String name) throws BackendException 
                             {
                                 Database db = environment.openDatabase(null, name, dbConfig);
                                 BerkeleyJEKeyValueStore store = new BerkeleyJEKeyValueStore(name, db, this);
                                 stores.put(name, store);
                             }
                         }
                         stores.put(name, store);
                     }
                     return stores.get(name);
                }
            }
            
            KeyColumnValueStore edgeStoreRaw = storeManagerLocking.openDatabase(EDGESTORE_NAME);
            {
                同上：  
                openDatabase("edgestore", EMPTY)
            }
            KeyColumnValueStore indexStoreRaw = storeManagerLocking.openDatabase(INDEXSTORE_NAME);
            {
                同上：  
                openDatabase("graphindex", EMPTY)
            }
            
            txLogManager.openLog(SYSTEM_TX_LOG_NAME);
            managementLogManager.openLog(SYSTEM_MGMT_LOG_NAME);
            txLogStore = new NoKCVSCache(storeManager.openDatabase(SYSTEM_TX_LOG_NAME));
            
            KeyColumnValueStore systemConfigStore = storeManagerLocking.openDatabase(SYSTEM_PROPERTIES_STORE_NAME);
            {
                同上：  
                openDatabase("system_properties", EMPTY)
            }
            
        }
        storeFeatures = backend.getStoreFeatures();
    }
    
    this.idAssigner = config.getIDAssigner(backend);
    this.idManager = idAssigner.getIDManager();
    
}

```

### management

```java
JanusGraphManagement management = graph.openManagement();
{
   new ManagementSystem(this,backend.getGlobalSystemConfig(),backend.getSystemMgmtLog(), managementLogger, schemaCache);
   //参数分别是 graph config Log managementLogger schemaCache
   {
       this.transaction = (StandardJanusGraphTx) graph.buildTransaction().disableBatchLoading().start();
       {
           graph.buildTransaction()
           {
               new StandardTransactionBuilder(getConfiguration(), this);
               {
                   
               }
           }
           disableBatchLoading()
           {
               
           }
           start()
           {
               new ImmutableTxCfg
               graph.newTransaction(immutable);
               {
                    StandardJanusGraphTx tx = new StandardJanusGraphTx(this, configuration);
                    {
                        父类： JanusGraphBlueprintsTransaction
                        太过复杂，跳过
                    }
                    tx.setBackendTransaction(openBackendTransaction(tx));
                    {
                        openBackendTransaction(tx)
                        {
                            IndexSerializer.IndexInfoRetriever retriever = indexSerializer.getIndexInfoRetriever(tx);
                            return backend.beginTransaction(tx.getConfiguration(), retriever);
                            {
                                StoreTransaction tx = storeManagerLocking.beginTransaction(configuration);
                                CacheTransaction cacheTx = new CacheTransaction(tx, storeManagerLocking, bufferSize, maxWriteTime, configuration.hasEnabledBatchLoading());
                                final Map<String, IndexTransaction> indexTx = new HashMap<>(indexes.size());
        						for (Map.Entry<String, IndexProvider> entry : indexes.entrySet()) {
        						    indexTx.put(entry.getKey(), new IndexTransaction(entry.getValue(), indexKeyRetriever.get(entry.getKey()), configuration, maxWriteTime));
        						}
        						return new BackendTransaction(cacheTx, configuration, storeFeatures,
                					edgeStore, indexStore, txLogStore,
                					maxReadTime, indexTx, threadPool);
                            }
                        }
                    }
                    openTransactions.add(tx);
                    return tx;
               }
           }
           
       }
   }
}

final PropertyKey name = management.makePropertyKey("name").dataType(String.class).make();
{
    management.makePropertyKey("name")
    {
        transaction.makePropertyKey(name);
        {
            new StandardPropertyKeyMaker(this, name, indexSerializer, attributeHandler);
            {
                super
                {
                    StandardRelationTypeMaker
                }
            }
        }
    }
    dataType(String.class)
    {
        dataType = clazz;
    }
    make();
    {
        new TypeDefinitionMap();
        tx.makePropertyKey(getName(), definition);
        {
            (PropertyKey) makeSchemaVertex(JanusGraphSchemaCategory.PROPERTYKEY, name, definition);
            {
                schemaVertex = new PropertyKeyVertex(this, IDManager.getTemporaryVertexID(IDManager.VertexIDType.UserPropertyKey, temporaryIds.nextID()), ElementLifeCycle.New);
                {
                    //一层层嵌套
                    
                }
            }
        }
    }
}

management.buildIndex("name", Vertex.class).addKey(name).unique().buildCompositeIndex();
{
    new IndexBuilder(indexName, ElementCategory.getByClazz(elementType));
    {
        
    }
    addKey(name)
    {
        keys.put(key, null);
    }
    unique()
    {
        unique = true;
    }
    buildCompositeIndex()
    {
        createCompositeIndex(indexName, elementCategory, unique, constraint, keyArr);
        {
            JanusGraphSchemaVertex indexVertex = transaction.makeSchemaVertex(JanusGraphSchemaCategory.GRAPHINDEX, indexName, def);
            {
                schemaVertex = new JanusGraphSchemaVertex(this, IDManager.getTemporaryVertexID(IDManager.VertexIDType.GenericSchemaType,temporaryIds.nextID()), ElementLifeCycle.New);
                {
                    //一层层嵌套
                    
                }
            }
            addSchemaEdge(indexVertex, keys[i], TypeDefinitionCategory.INDEX_FIELD, paras);
            
            updateSchemaVertex(indexVertex);
            JanusGraphIndexWrapper index = new JanusGraphIndexWrapper(indexVertex.asIndexType());
            updateIndex(index, SchemaAction.REGISTER_INDEX);
            return index;
        }
    }
    
}
```

### containsVertexLabel

mgmt.getVertexLabels().iterator()
mgmt.containsVertexLabel(label)
这两个方法都可以得到 VertexLABEL

首先看 mgmt.getVertexLabels().iterator(), 这里面首先通过了 guava 的 abstractIterator 转到一个 ResultSetIterator

```java

public ResultSetIterator(Iterator<R> inner, int limit) {
    this.iter = inner;
    this.limit = limit;
    count = 0;
    this.current = null;
    this.next = nextInternal();
    {
        QueryProcessor$LimitAdajustingIterator.hasNext()
        {
            ....省去一步调用
            executor.execute(query, backendQuery, executionInfo, profiler);
            {
                iter = new SubqueryIterator(indexQuery.getQuery(0), indexSerializer, txHandle, indexCache, indexQuery.getLimit(), getConversionFunction(query.getResultType()),
                        retrievals.isEmpty() ? null: QueryUtil.processIntersectingRetrievals(retrievals, indexQuery.getLimit()));
                {
                    stream = indexSerializer.query(subQuery, tx).map(r -> {
                        currentIds.add(r);
                        return r;
                    });
                    {
                        final List<EntryList> rs = sq.execute(tx);
                        {
                            EntryList next =tx.indexQuery(ksq.updateLimit(getLimit()-total));
                            {
                                return exe.call();
                                {
                                    return cacheEnabled?indexStore.getSlice(query, storeTx):
                                        indexStore.getSliceNoCache(query, storeTx);
                                    {
                                        CassandraThriftKeyColumnValueStore.getNamesSlice(ImmutableList.of(key),query,txh);
                                    }
                                }
                            }
                        }
                        
                    }
                }
            }
        }
        
    }
}
```
这上面已经是省略很多步骤的调用栈。。。

mgmt.containsVertexLabel(label) 调用栈稍微少了一点：

```java
JanusGraphSchemaVertex getSchemaVertex(String schemaName)
{
    id = retriever.retrieveSchemaByName(schemaName);
    {
        JanusGraphVertex v = Iterables.getOnlyElement(QueryUtil.getVertices(consistentTx, BaseKey.SchemaName, typeName), null);
        {
            new ResultSetIterator()
            {
                ....
                runWithMetrics
                iter = new SubqueryIterator(indexQuery.getQuery(0), indexSerializer, txHandle, indexCache, indexQuery.getLimit(), getConversionFunction(query.getResultType()),
                        retrievals.isEmpty() ? null: QueryUtil.processIntersectingRetrievals(retrievals, indexQuery.getLimit()));
                {
                    类似上面
                }
            }
        }
    }
}
```

### makeVertexLabel

```java
mgmt.makeVertexLabel(vType.toString()).make();
{
    StandardVertexLabelMaker.make
    return (VertexLabelVertex)tx.makeSchemaVertex(JanusGraphSchemaCategory.VERTEXLABEL,name,def);
    {
        
        public final JanusGraphSchemaVertex makeSchemaVertex(JanusGraphSchemaCategory schemaCategory, String name, TypeDefinitionMap definition) 
        {
            1. new VertexLabelVertex
            schemaVertex = new VertexLabelVertex(this, IDManager.getTemporaryVertexID(IDManager.VertexIDType.GenericSchemaType,temporaryIds.nextID()), ElementLifeCycle.New);
            2. graph.assignID(schemaVertex, BaseVertexLabel.DEFAULT_VERTEXLABEL);
            
            3. addProperty(schemaVertex, BaseKey.SchemaName, schemaCategory.getSchemaName(name));
            
            4. updateSchemaVertex(schemaVertex);
        }
    }
}
```

assignID应该是 生产者消费者模式。
```java
IDBlock idBlock = idAuthority.getIDBlock(partition, idNamespace, renewTimeout);
{
    long nextStart = getCurrentID(partitionKey);
    {
        ......
        return idStore.getSlice(new KeySliceQuery(partitionKey, LOWER_SLICE, UPPER_SLICE).setLimit(5), txh);
    }
}
```




### containsPropertyKey

### makePropertyKey

### containsEdgeLabel

### makeEdgeLabel

基本上和上面类似，接下来深入分析一下这些调用栈涉及到的类。
