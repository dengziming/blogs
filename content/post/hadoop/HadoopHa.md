---
date: 2018-05-23
title: "resourcemanager"
author: "邓子明"
tags:
    - hadoop
    - 源码
categories:
    - hadoop源码
comment: true
---


参考资料：

#        HadoopHa

hadoop 有两个NameNode，Active NameNode和Standby NameNode，通过 DFSZKFailoverController extends ZKFailoverController 进行切换。
ZKFailoverController通过HealthMonitor线程能及时检测到NameNode的健康状况，在主NameNode故障时借助Zookeeper实现自动的主备选举和切换。
DataNode 会同时向主NameNode和备NameNode上报数据块的位置信息，但只接收来自active namenode的读写命令。

为啥把监控分开?

显然，我们不能在NN进程内进行心跳等信息同步，最简单的原因，一次FullGC就可以让NN挂起十几分钟，所以，必须要有一个独立的短小精悍的watchdog来专门负责监控。这也是一个松耦合的设计，便于扩展或更改。

通过隔离和Quorum Journal Manager(QJM)共享存储空间实现HDFS HA

# DFSZKFailoverController

启动代码：
```java
  public static void main(String args[])
      throws Exception {
    if (DFSUtil.parseHelpArgument(args, 
        ZKFailoverController.USAGE, System.out, true)) {
      System.exit(0);
    }
    
    GenericOptionsParser parser = new GenericOptionsParser(
        new HdfsConfiguration(), args);
    DFSZKFailoverController zkfc = DFSZKFailoverController.create(
        parser.getConfiguration());
    {
        NNHAServiceTarget localTarget = new NNHAServiceTarget(
        localNNConf, nsId, nnId);
        return new DFSZKFailoverController(localNNConf, localTarget);
    }
    
    System.exit(zkfc.run(parser.getRemainingArgs()));
  }
```

run 的步骤：
initZK();
formatZK(force, interactive);
initRPC();
initHM();
startRPC();
mainLoop();

```java
initZK();
{
    elector = new ActiveStandbyElector(zkQuorum,
        zkTimeout, getParentZnode(), zkAcls, zkAuths,
        new ElectorCallbacks(), maxRetryNum);
    {
    	new ElectorCallbacks()
    	  // 临时节点ActiveStandbyElectorLock，用于标识锁
    	zkLockFilePath = znodeWorkingDir + "/" + LOCK_FILENAME;
    	// 永久节点ActiveBreadCrumb，用于存放active信息
    	zkBreadCrumbPath = znodeWorkingDir + "/" + BREADCRUMB_FILENAME;
    	this.maxRetryNum = maxRetryNum;
    	// createConnection for future API calls
    	// 创建zk连接
    	createConnection();
    	{
    	      // 不幸的是，zk的构造方法连接上zk之后，可能马上触发连接事件。
  			  // 因此如果构造zk之后注册watcher，可能不会捕获到连接事件。
  			  // 取而代之的方法是，先构造Watcher，在设置了zk的引用之前，使它阻塞所有的事件
  			  
  			  watcher = new WatcherWithClientRef();
  			  ZooKeeper zk = new ZooKeeper(zkHostPort, zkSessionTimeout, watcher);
  			  // 在watcher中设置zk的引用
  			  watcher.setZooKeeperRef(zk);
  			  // Wait for the asynchronous success/failure. This may throw an exception
  			  // if we don't connect within the session timeout.
  			  watcher.waitForZKConnectionEvent(zkSessionTimeout);
  			  
  			  for (ZKAuthInfo auth : zkAuthInfo) {
  			    zk.addAuthInfo(auth.getScheme(), auth.getAuth());
  			  }
  			  return zk;
    	      }
    	  }
    	}
}

formatZK(force, interactive);
initRPC();
{
    new ZKFCRpcServer(conf, bindAddr, this, getPolicyProvider());
    {
          this.zkfc = zkfc;
  			// 使用protocol buffer序列化
  			RPC.setProtocolEngine(conf, ZKFCProtocolPB.class,
  			    ProtobufRpcEngine.class);
  			ZKFCProtocolServerSideTranslatorPB translator =
  			    new ZKFCProtocolServerSideTranslatorPB(this);
  			BlockingService service = ZKFCProtocolService
  			    .newReflectiveBlockingService(translator);
  			// 使用hadoop rpc接口得到rpc server
  			// ZKFCProtocol是rpc协议，service是rpc协议的实现类
  			// ZKFCProtocolPB是protobuf rpc接口的一个过渡类
  			this.server = new RPC.Builder(conf).setProtocol(ZKFCProtocolPB.class)
  			    .setInstance(service).setBindAddress(bindAddr.getHostName())
  			    .setPort(bindAddr.getPort()).setNumHandlers(HANDLER_COUNT)
  			    .setVerbose(false).build();
    }
}
initHM();
startRPC();
mainLoop();
```


WatcherWithClientRef 在构造zk时被注册为默认watcher，主要监听连接或者断开事件。当调用initZk之后，watcher.process会对事件进行处理，连接、断开、过期的状态类型都是EventType.None。