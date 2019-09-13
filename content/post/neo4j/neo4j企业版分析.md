---
date: 2018-03-22
title: "neo4j企业版分析"
author: "邓子明"
tags:
    - neo4j
    - 图数据库
categories:
    - 源码分析
comment: true
---

阅读neo4j源码是为了改造，所以研究一下企业版的源码。OpenEnterpriseNeoServer 和 CommunityNeoServer 稍微对比一下。

CommunityNeoServer
```java
protected static final GraphFactory COMMUNITY_FACTORY = ( config, dependencies ) ->
    {
        File storeDir = config.get( GraphDatabaseSettings.database_path );
        return new GraphDatabaseFacadeFactory( DatabaseInfo.COMMUNITY, CommunityEditionModule::new )
                .newFacade( storeDir, config, dependencies );
    };

    public CommunityNeoServer( Config config, GraphDatabaseFacadeFactory.Dependencies dependencies,
            LogProvider logProvider )
    {
        this( config, lifecycleManagingDatabase( COMMUNITY_FACTORY ), dependencies, logProvider );
    }
```

OpenEnterpriseNeoServer
```java
    protected static Database.Factory createDbFactory( Config config )
    {
        final Mode mode = config.get( EnterpriseEditionSettings.mode );

        switch ( mode )
        {
        case HA:
            return lifecycleManagingDatabase( HA_FACTORY );
        case ARBITER:
            // Should never reach here because this mode is handled separately by the scripts.
            throw new IllegalArgumentException( "The server cannot be started in ARBITER mode." );
        case CORE:
            return lifecycleManagingDatabase( CORE_FACTORY );
        case READ_REPLICA:
            return lifecycleManagingDatabase( READ_REPLICA_FACTORY );
        default:
            return lifecycleManagingDatabase( ENTERPRISE_FACTORY );
        }
    }
    
    private static final GraphFactory HA_FACTORY = ( config, dependencies ) ->
    {
        File storeDir = config.get( GraphDatabaseSettings.database_path );
        return new HighlyAvailableGraphDatabase( storeDir, config, dependencies );
    };

    private static final GraphFactory ENTERPRISE_FACTORY = ( config, dependencies ) ->
    {
        File storeDir = config.get( GraphDatabaseSettings.database_path );
        return new EnterpriseGraphDatabase( storeDir, config, dependencies );
    };

    private static final GraphFactory CORE_FACTORY = ( config, dependencies ) ->
    {
        File storeDir = config.get( GraphDatabaseSettings.database_path );
        return new CoreGraphDatabase( storeDir, config, dependencies );
    };

    private static final GraphFactory READ_REPLICA_FACTORY = ( config, dependencies ) ->
    {
        File storeDir = config.get( GraphDatabaseSettings.database_path );
        return new ReadReplicaGraphDatabase( storeDir, config, dependencies );
    };
    
```

最终不一样的还是启动的 server的不一样而已。最终是在 PlatformModule EditionModule DataSourceModule 三个类负责的 中不一样的。

