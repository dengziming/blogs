---
date: 2018-03-22
title: "neo4j源码分析2-启动源码跟踪"
author: "邓子明"
tags:
    - neo4j
    - 图数据库
categories:
    - 源码分析
comment: true
---

## 1.第一遍调试
第一遍就是打断点，然后查看调用栈，忽略过多的线程。

找到 CommunityEntryPoint，打一个断点，调试,不断F5进入，F6单步执行，F跳出。
1. `new CommunityBootstrapper(),ServerBootstrapper.start(boot,args)`

2. `ServerBootstrapper` 中的初始化关键代码： `private GraphDatabaseDependencies dependencies = GraphDatabaseDependencies.newDependencies()`; 这个dependencies貌似来头很大。F5进入
    `public static GraphDatabaseDependencies newDependencies()`
    1. `KernelExtensionFactory factory : Service.load( KernelExtensionFactory.class) `
        这段代码似乎跳不进去，反正最后得到了7个:

		0 = {LuceneKernelExtensionFactory@675} "KernelExtension:LuceneKernelExtensionFactory[lucene]"
		1 = {LuceneSchemaIndexProviderFactory@679} "KernelExtension:LuceneSchemaIndexProviderFactory[lucene]"
		2 = {NativeLuceneFusionSchemaIndexProviderFactory@680} "KernelExtension:NativeLuceneFusionSchemaIndexProviderFactory[lucene+native]"
		3 = {BoltKernelExtension@681} "KernelExtension:BoltKernelExtension[bolt-server]"
		4 = {ShellServerExtensionFactory@682} "KernelExtension:ShellServerExtensionFactory[shell]"
		5 = {UdcKernelExtensionFactory@683} "KernelExtension:UdcKernelExtensionFactory[kernel udc]"
		6 = {JmxExtensionFactory@684} "KernelExtension:JmxExtensionFactory[kernel jmx]"


	2. `List<QueryEngineProvider> queryEngineProviders = asList( Service.load( QueryEngineProvider.class ) );`

	    这段代码和前面一样，不过加载的是查询引擎的的class，我们暂且跳过！

    3. `return new GraphDatabaseDependencies( null, null, new ArrayList<>(), kernelExtensions,)`

3. `ServerBootstrapper.start( Bootstrapper boot, String... argv )`
```java
    1. `CommunityBootstrapper(AbstractNeoServer).start`

        1. `server = createNeoServer( config, dependencies, userLogProvider );`

            1. `new CommunityNeoServer( config, dependencies, logProvider );`
            
            
            // 初始化很多属性

    		protected abstract WebServer createWebServer();

			// 放在代码后面的属性
			private final Dependencies dependencyResolver = new Dependencies( new Supplier<DependencyResolver>()
    		{
    		    @Override
    		    public DependencyResolver get()
    		    {
    		        Database db = dependencyResolver.resolveDependency( Database.class );
    		        return db.getGraph().getDependencyResolver();
    		    }
    		} );
			// 构造方法
    		public AbstractNeoServer( Config config, Database.Factory dbFactory,
    		        GraphDatabaseFacadeFactory.Dependencies dependencies, LogProvider logProvider )
    		{
    		    this.logProvider = logProvider;
    		    // 初始化很多东西
    		}
            

        2. `AbstractNeoServer.start();`

            1. `init()`

                1. `this.database = life.add( dependencyResolver.satisfyDependency( dbFactory.newDatabase( config, dependencies ) ) );`

                    1. `lambda$lifecycleManagingDatabase$0:47, LifecycleManagingDatabase (org.neo4j.server.database)`
                    这里的 java8 lamabda表达式有点不懂，总之就是这个 dbFactory.newDatabase( config, dependencies ) 执行的是这段代码：
                     `( config, dependencies ) -> new LifecycleManagingDatabase( config, graphDbFactory, dependencies );`

                    2. `dependencyResolver.satisfyDependency(LifecycleManagingDatabase )`
                    这里的 satisfyDependency 方法有点奇怪，总之就是将 LifecycleManagingDatabase 的所有父类添加到一个临时变量，好像啥也没做。

                    3. `life.add(LifecycleManagingDatabase)`
                    奇怪的代码，后续我们专门讲解这个 life 的实现，这是注释：
                    Add a new Lifecycle instance. It will immediately be transitioned to the state of this LifeSupport.
                    将传入的dependency 新建为一个LifecycleInstance，add到 instances中。
                    
                    LifecycleInstance newInstance = new LifecycleInstance( instance );
                    private volatile List<LifecycleInstance> instances = new ArrayList<>();

                    4. `this.database = LifecycleManagingDatabase`
                
                2. 新建其他的，非内核部分我们忽略。

                
                this.authManagerSupplier = dependencyResolver.provideDependency( AuthManager.class );
        		this.userManagerSupplier = dependencyResolver.provideDependency( UserManagerSupplier.class );
        		this.sslPolicyFactorySupplier = dependencyResolver.provideDependency( SslPolicyLoader.class );
        		this.webServer = createWebServer();
                

                3. createServerModules()
                
                
                return Arrays.asList(
                new DBMSModule( webServer, getConfig() ),
                new RESTApiModule( webServer, getConfig(), getDependencyResolver(), logProvider ),
                new ManagementApiModule( webServer, getConfig() ),
                new ThirdPartyJAXRSModule( webServer, getConfig(), logProvider, this ),
                new ConsoleModule( webServer, getConfig() ),
                new Neo4jBrowserModule( webServer ),
                createAuthorizationModule(),
                new SecurityRulesModule( webServer, getConfig(), logProvider ) );
                

                4.创建 ServerComponentsLifecycleAdapter

                
                serverComponents = new ServerComponentsLifecycleAdapter();
                life.add( serverComponents );

                this.initialized = true;
                

            2. life.start();
            debug进入：`LifeSupport`

                1. init();
                    1. status = changedStatus( this, status, LifecycleStatus.INITIALIZING );

                    2. for ( LifecycleInstance instance : instances ) instance.init();
                        这时候有两个instance，分别是上面我们new出来的  new LifecycleManagingDatabase( config, graphDbFactory, dependencies ) 和 new ServerComponentsLifecycleAdapter();

                        1. LifecycleInstance.init()
                        还好两个代码里面什么都没做，不然再F5进去，我要奔溃了。。。

                    3. status = changedStatus( this, status, LifecycleStatus.STOPPED );

                2. for ( LifecycleInstance instance : instances ) instance.start();
                    这时候有两个instance，分别是上面我们new出来的  new LifecycleManagingDatabase( config, graphDbFactory, dependencies ) 和 new ServerComponentsLifecycleAdapter();

                        1. `LifecycleInstance.start() `

                        2. `LifecycleManagingDatabase.start()`

                            
                            log.info( "Starting..." );
        					this.graph = dbFactory.newGraphDatabase( config, dependencies );
        					if ( !isInTestMode() )
        					{
        					    preLoadCypherCompiler();
        					}
        					log.info( "Started." );

        					

        					1. this.graph = dbFactory.newGraphDatabase( config, dependencies );
        					这里又是lambda表达式：
        					new GraphDatabaseFacadeFactory,
        					
        					File storeDir = config.get( GraphDatabaseSettings.database_path );
                            return new GraphDatabaseFacadeFactory( DatabaseInfo.COMMUNITY, CommunityEditionModule::new )
                                   .newFacade( storeDir, config, dependencies );
                            
                            主要的核心代码已经找到了：new GraphDatabaseFacadeFactory( DatabaseInfo.COMMUNITY, CommunityEditionModule::new ).newFacade( storeDir, config, dependencies );
                            接下来我们主要调试这一段。

                        3. `ServerComponentsLifecycleAdapter.start()`
                            这里主要是和web，cypher有关，我们暂时忽略。

```


## 2.调试 GraphDatabaseFacadeFactory.newFacade
上面我们已经调试到了 最后的部分，也是高潮部分。

newFacade 方法：
1. `initFacade( storeDir, config, dependencies, new GraphDatabaseFacade() );`

```java
    1.`new GraphDatabaseFacade()`
    初始化相关数据

    2.`GraphDatabaseFacade initFacade( File storeDir, Config config, final Dependencies dependencies, final GraphDatabaseFacade graphDatabaseFacade )`

    	1. `PlatformModule platform = createPlatform( storeDir, config, dependencies, graphDatabaseFacade );`

        	1. `new PlatformModule( storeDir, config, databaseInfo, dependencies, graphDatabaseFacade );`
            这一部分代码很长很关键的感觉，这里是内核相关，先跳过，回头看。下一章节 TODO

            2. EditionModule edition = editionFactory.apply( platform );
            
            这里和上一个PlatformModule干的事情一样。下一章节 TODO

        2. `final DataSourceModule dataSource = createDataSource( platform, edition, queryEngine::get );`

            1. new DataSourceModule( platformModule, editionModule, queryEngine );
            和上面的 PlatformModule 一样，一大堆的新建。。。最后 life.add( platformModule.kernelExtensions ) 新建DataSource
        3. `ClassicCoreSPI spi = new ClassicCoreSPI( platform, dataSource, msgLog, coreAPIAvailabilityGuard );`
        
        4. `platform.life.start();`
 
            1. init();
            2. for ( LifecycleInstance instance : instances ) instance.start();
                这里的instance ：
           		
           		
           		0 = {LifeSupport$LifecycleInstance@3583} "org.neo4j.io.fs.FileSystemLifecycleAdapter@353d0772: STARTED"
		 		1 = {LifeSupport$LifecycleInstance@3730} "org.neo4j.kernel.impl.util.Neo4jJobScheduler@2667f029: STARTED"
		 		2 = {LifeSupport$LifecycleInstance@3635} "org.neo4j.udc.UsageData@67a20f67: STARTED"
		 		3 = {LifeSupport$LifecycleInstance@3658} "org.neo4j.kernel.impl.logging.StoreLogService@57c758ac: STARTED"
		 		4 = {LifeSupport$LifecycleInstance@3681} "org.neo4j.kernel.internal.locker.StoreLockerLifecycleAdapter@a9cd3b1: STARTED"
		 		5 = {LifeSupport$LifecycleInstance@3731} "org.neo4j.kernel.impl.pagecache.PageCacheLifecycle@13e39c73: STOPPED"
		 		6 = {LifeSupport$LifecycleInstance@3732} "org.neo4j.kernel.info.DiagnosticsManager@64cd705f: STOPPED"
		 		7 = {LifeSupport$LifecycleInstance@3733} "org.neo4j.kernel.impl.transaction.state.DataSourceManager@548d708a: STOPPED"
		 		8 = {LifeSupport$LifecycleInstance@3734} "org.neo4j.kernel.impl.util.watcher.DefaultFileSystemWatcherService@4b013c76: STOPPED"
		 		9 = {LifeSupport$LifecycleInstance@3735} "org.neo4j.kernel.impl.core.DelegatingPropertyKeyTokenHolder@53fb3dab: STOPPED"
		 		10 = {LifeSupport$LifecycleInstance@3736} "org.neo4j.kernel.impl.core.DelegatingLabelTokenHolder@cb0755b: STOPPED"
		 		11 = {LifeSupport$LifecycleInstance@3737} "org.neo4j.kernel.impl.core.DelegatingRelationshipTypeTokenHolder@33065d67: STOPPED"
		 		12 = {LifeSupport$LifecycleInstance@3738} "org.neo4j.kernel.internal.DefaultKernelData@30: STOPPED"
		 		13 = {LifeSupport$LifecycleInstance@3739} "org.neo4j.kernel.impl.core.ThreadToStatementContextBridge@7bba5817: STOPPED"
		 		14 = {LifeSupport$LifecycleInstance@3740} "org.neo4j.kernel.extension.KernelExtensions@25df00a0: STOPPED"
		 		15 = {LifeSupport$LifecycleInstance@3741} "org.neo4j.kernel.impl.proc.Procedures@6cc4cdb9: STOPPED"
		 		16 = {LifeSupport$LifecycleInstance@3742} "org.neo4j.server.security.auth.BasicAuthManager@47c81abf: STOPPED"
		 		17 = {LifeSupport$LifecycleInstance@3743} "org.neo4j.kernel.impl.cache.MonitorGc@30b6ffe0: STOPPED"
		 		18 = {LifeSupport$LifecycleInstance@3744} "org.neo4j.kernel.impl.pagecache.PublishPageCacheTracerMetricsAfterStart@2415fc55: STOPPED"
		 		19 = {LifeSupport$LifecycleInstance@3745} "org.neo4j.kernel.DatabaseAvailability@1890516e: STOPPED"
		 		20 = {LifeSupport$LifecycleInstance@3746} "org.neo4j.kernel.impl.factory.DataSourceModule$StartupWaiter@16c069df: STOPPED"
		 		21 = {LifeSupport$LifecycleInstance@3747} "org.neo4j.kernel.internal.KernelEventHandlers@2bec854f: STOPPED"
           	   
               每个instance的start方法具体是怎样的，我们稍后细看，这里跳过 TODO

```

## 3.学习neo4j server的设计模式
上面我们调试了一遍启动过程，整个个过程可以多来几次，每一遍加深对neo4j的理解。
调试之前我们学习一下 LifeSupport 这个类的设计和使用。

LifeSupport继承自Lifecycle，源码如下：
```java
/**
 * Lifecycle interface for kernel components. Init is called first,
 * followed by start,
 * and then any number of stop-start sequences,
 * and finally stop and shutdown.
 *
 * As a stop-start cycle could be due to change of configuration, please perform anything that depends on config
 * in start().
 *
 * Implementations can throw any exception. Caller must handle this properly.
 *
 * The primary purpose of init in a component is to set up structure: instantiate dependent objects,
 * register handlers/listeners, etc.
 * Only in start should the component actually do anything with this structure.
 * Stop reverses whatever was done in start, and shutdown finally clears any set-up structure, if necessary.
 */
public interface Lifecycle
{
    void init() throws Throwable;

    void start() throws Throwable;

    void stop() throws Throwable;

    void shutdown() throws Throwable;

}
```
注释很清楚，万一看不懂百度翻译一下就明白。注意这里：init只是set up structure——初始化依赖的对象，注册处理器/监听器。只有start方法执行后才会用这个structure TOTDO，是不是看源码可以跳过init

按F4发现有很多的实现类
```java
PaxosClusterMemberAvailability (org.neo4j.cluster.member.paxos)
DefaultKernelData (org.neo4j.kernel.internal)
LifecycleAdapter (org.neo4j.kernel.lifecycle)
NeoStoreDataSource (org.neo4j.kernel)
TransactionPropagator (org.neo4j.kernel.ha.transaction)
ShellServerKernelExtension (org.neo4j.shell.impl)
OnlineBackupKernelExtension (org.neo4j.backup)
DummyExtension (org.neo4j.kernel)
LifeSupport (org.neo4j.kernel.lifecycle)
HighAvailabilityModeSwitcher (org.neo4j.kernel.ha.cluster.modeswitch)
KernelEventHandlers (org.neo4j.kernel.internal)
RecordStorageEngine (org.neo4j.kernel.impl.storageengine.impl.recordstorage)
JmxKernelExtension (org.neo4j.jmx.impl)
ExecutorLifecycleAdapter (org.neo4j.cluster)
KernelExtensions (org.neo4j.kernel.extension)
RecoveryCleanupWorkCollector (org.neo4j.index.internal.gbptree)
IndexImplementation (org.neo4j.kernel.spi.explicitindex)
NetworkReceiver (org.neo4j.cluster.com)
...
```

然后我们重点看看 LifeSupport ，我们分析发现 LifeSupport 也是一个 Lifecycle，而且有一个 LifecycleInstance 的数组 instances ，

LifecycleInstance 也是继承自 Lifecycle。所以实际上 LifeSupport 就是一堆 Lifecycle 放在了一起，进行了一个类似装饰模式而已。

LifeSupport的init,start,stop,shutdown方法，分别是循环instances执行init,start,stop,shutdown方法。

经过上面的调试，我们发现neo4j基本上就是一个一个这样的 LifeSupport 组成的。

### (1). 第一次使用
我们第一次遇到 LifeSupport是 在： CommunityBootstrapper.start() 时候，先创建 CommunityNeoServer，调用它的 start，start 前先是init方法。遇到了两个代码：

```java
this.database = life.add( dependencyResolver.satisfyDependency( dbFactory.newDatabase( config, dependencies ) ) );
serverComponents = new ServerComponentsLifecycleAdapter();
life.add( serverComponents );
```

这里的 life 就是 AbstractNeoServer(CommunityNeoServer) LifeSupport，是父类的成员变量，新建 CommunityNeoServer 的时候初始化的。然后在init方法中给他添加了两个 Lifecycle 的实现对象。

AbstractNeoServer(CommunityNeoServer)执行完了init方法，就执行 life 的start方法，实际上执行的还是 new LifecycleManagingDatabase( config, graphDbFactory, dependencies ) 和 new ServerComponentsLifecycleAdapter() 的start。

总结：
CommunityNeoServer 中的 出现的 LifeSupport 为：

1. new LifecycleManagingDatabase( config, graphDbFactory, dependencies )
 这里的 graphDbFactory 是 CommunityNeoServer 的一个匿名内部类接口的实现类，dependencies就是包含了 kernelExtensions 等内容的东西。


2. new ServerComponentsLifecycleAdapter()

### (2). 第二次使用

我们后面还有一次用到了 LifeSupport 。就是执行life的start时候，需要上面的 LifecycleManagingDatabase 的 start 方法。里面最重要的就是 dbFactory.newGraphDatabase( config, dependencies );

这个 dbFactory 我们已经说了是 CommunityNeoServer 的一个匿名内部类接口的实现类 COMMUNITY_FACTORY， 最终执行方法返回：

`return new GraphDatabaseFacadeFactory( DatabaseInfo.COMMUNITY, CommunityEditionModule::new ).newFacade( storeDir, config, dependencies );`

TODO 这里出现了 DatabaseInfo.COMMUNITY ，如果我们想使用企业版的功能，肯定需要在这里修改源码。还有 CommunityEditionModule ，
创建的实例只能用于社区版，所以是否可以猜想，企业版就是比社区版多了几个 LifeSupport 而已

然后调用 GraphDatabaseFacadeFactory的newFacade方法，`return initFacade( storeDir, config, dependencies, new GraphDatabaseFacade() );`

这里 new GraphDatabaseFacade(),然后初始化，实际上就是初始化一个数据库了。

然后创建一个 platform ，经过一堆复杂处理后，调用 platform 的 life 的start方法。也就是我们关心的 LifeSupport 。在查看之前，我们需要知道创建这个 platform 干了啥。

打开PlatformModule的构造方法，太复杂了。。。。，但是先别泄气，我们先抓 LifeSupport 吧，搞定了这个再看别的。


前面几行 F6 跳过，然后直接看：`life = dependencies.satisfyDependency( createLife() );`
这个 createLife 方法就是new了一个 LifeSupport，然后F6跳过几行，直接看：
```java
fileSystem = dependencies.satisfyDependency( createFileSystemAbstraction() );
life.add( new FileSystemLifecycleAdapter( fileSystem ) );
```
这里 createFileSystemAbstraction 就是 new DefaultFileSystemAbstraction，然后添加到 life，这时候 life 的 size 已经是1了。

然后F6跳过几行，直接看： `jobScheduler = life.add( dependencies.satisfyDependency( createJobScheduler() ) );`

这里 createJobScheduler 就是 Neo4jJobScheduler。这时候 life 的 size 已经是2了。

然后F6跳过几行，直接看： `dependencies.satisfyDependency( life.add( new UsageData( jobScheduler ) ) );`

这里 new UsageData ,这时候 life 的 size 已经是3了。

然后F6跳过几行，直接看： `life.add( dependencies.satisfyDependency( new StoreLockerLifecycleAdapter( createStoreLocker() ) ) );`

这里的 createStoreLocker 就是 new GlobalStoreLocker( fileSystem, storeDir );

然后 new StoreLockerLifecycleAdapter，这时候 life 的 size 已经是5了。我很好奇为啥突然加了两个。多了一个 StoreLogservice


然后F6跳过几行，直接看：
`pageCache = dependencies.satisfyDependency( createPageCache( fileSystem, config, logging, tracers ) );life.add( new PageCacheLifecycle( pageCache ) );`

然后 new PageCacheLifecycle( pageCache ) 这时候 life 的 size 已经是6了。


继续查看：`diagnosticsManager = life.add( dependencies.satisfyDependency( new DiagnosticsManager( logging.getInternalLog( DiagnosticsManager.class ) ) ) );`
这里 new DiagnosticsManager( logging.getInternalLog( DiagnosticsManager.class ) )  ，这时候 life 的 size 已经是7了。


一直到 createPlatform 运行完，life一共有7个 LifeSupport。然后调用：`EditionModule edition = editionFactory.apply( platform );`直接进入 CommunityEditionModule 的构造方法


直接F6: `LifeSupport life = platformModule.life;life.add( platformModule.dataSourceManager );`
这里添加了 dataSourceManager，实际上是个 DAtaSourceManager。这时候 life 的 size 已经是8了。
```java
life.add( watcherService );
propertyKeyTokenHolder = life.add( dependencies.satisfyDependency( new DelegatingPropertyKeyTokenHolder(
createPropertyKeyCreator( config, dataSourceManager, idGeneratorFactory ) ) ) );
labelTokenHolder = life.add( dependencies.satisfyDependency(new DelegatingLabelTokenHolder( createLabelIdCreator( config,
dataSourceManager, idGeneratorFactory ) ) ));
relationshipTypeTokenHolder = life.add( dependencies.satisfyDependency(new DelegatingRelationshipTypeTokenHolder(
createRelationshipTypeCreator( config, dataSourceManager, idGeneratorFactory ) ) ));
```
这时候 life 的 size 已经是12了。


`dependencies.satisfyDependency(createKernelData( fileSystem, pageCache, storeDir, config, graphDatabaseFacade, life ) );`

这个方法运行完成后，life一共有12个 LifeSupport。

然后是：`final DataSourceModule dataSource = createDataSource( platform, edition, queryEngine::get );`
这里我们只抓取和life有关的代码：
```java
LifeSupport life = platformModule.life;
threadToTransactionBridge = deps.satisfyDependency( life.add( new ThreadToStatementContextBridge() ) );
life.add( platformModule.kernelExtensions );
```
和

```java

life.add( new MonitorGc( config, logging.getInternalLog( MonitorGc.class ) ) );

life.add( new PublishPageCacheTracerMetricsAfterStart( platformModule.tracers.pageCursorTracerSupplier ) );

life.add( new DatabaseAvailability( platformModule.availabilityGuard, platformModule.transactionMonitor,
        config.get( GraphDatabaseSettings.shutdown_transaction_end_timeout ).toMillis() ) );

life.add( new StartupWaiter( platformModule.availabilityGuard, editionModule.transactionStartTimeout ) );

// Kernel event handlers should be the very last, i.e. very first to receive shutdown events
life.add( kernelEventHandlers );
```

到这里life的size已经是22 。

然后会调用platform.life.start().就会循环调用上面的所有的 LifeSupport 的start方法。

所以实际上，整个代码的运行就是一个个的 lifeSupport 的运行，

## 4.理解LifeSupport后再次调试代码

这次调试就好多了，我们可以着重看重要的代码

```java
1. `server = createNeoServer( config, dependencies, userLogProvider );`

    1. `this( config, lifecycleManagingDatabase( COMMUNITY_FACTORY ), dependencies, logProvider );`

        1.
        
        COMMUNITY_FACTORY = ( config, dependencies ) ->
    	{
    	    File storeDir = config.get( GraphDatabaseSettings.database_path );
    	    return new GraphDatabaseFacadeFactory( DatabaseInfo.COMMUNITY, CommunityEditionModule::new )
    	            .newFacade( storeDir, config, dependencies );
    	};
        

2. `server.start();`

    1. `init`

        1. `this.database = life.add( dependencyResolver.satisfyDependency( dbFactory.newDatabase( config, dependencies ) ) );`

            1. `new GraphDatabaseFacadeFactory( DatabaseInfo.COMMUNITY, CommunityEditionModule::new ).newFacade( storeDir, config, dependencies );`

                1. `life.add(GraphDatabaseFacadeFactory)`

            2. `this.webServer = createWebServer();`

                1. `new JettyWebServer()`

            3. `createServerModules()`

            4. `serverComponents = new ServerComponentsLifecycleAdapter();`

    2. `life.start();`

        1. for ( LifecycleInstance instance : instances ) start()
```

上面两个 Lifecycle start 分开看。

1. `LifecycleManagingDatabase.start()`

    1. `this.graph = dbFactory.newGraphDatabase( config, dependencies );`

        1. `return new GraphDatabaseFacadeFactory( DatabaseInfo.COMMUNITY, CommunityEditionModule::new ).newFacade( storeDir, config, dependencies );`

            1. `new GraphDatabaseFacadeFactory( DatabaseInfo.COMMUNITY, CommunityEditionModule::new )` 这里的lambda类似scala的匿名函数，钩子方法。

            2. `newFacade( File storeDir, Config config, final Dependencies dependencies )`

                1. `new GraphDatabaseFacade()`

                2. `initFacade( File storeDir, Config config, final Dependencies dependencies,final GraphDatabaseFacade graphDatabaseFacade )`

                    1. `PlatformModule platform = createPlatform( storeDir, config, dependencies, graphDatabaseFacade );`
                    这里就是上面我们省略的部分，

                    2. `EditionModule edition = editionFactory.apply( platform );`

                    3. `final DataSourceModule dataSource = createDataSource( platform, edition, queryEngine::get );`

                    4. `platform.life.start();`
                    start的过程启动了所有的 LifeCycle
2. `ServerComponentsLifecycleAdapter.start()`
后续的程序

接下来我们就是一个一个分析

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
