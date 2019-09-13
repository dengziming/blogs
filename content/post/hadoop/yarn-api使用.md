---
date: 2018-05-23
title: "yarn-api使用"
author: "邓子明"
tags:
    - hadoop
    - 源码
categories:
    - hadoop源码
comment: true
---

参考： https://ieevee.com/tech/2015/05/05/yarn-dist-shell.html

# distributeshell

##  Client解析

distShell主要有2个类组成，Client和ApplicationMaster。两个类都带有main入口。Client的主要工作是启动AM，真正要做的任务由AM来调度。 Client的简化框架如下。

```java
public static void main(String[] args) {
    boolean result = false;
    try {
      Client client = new Client();  //1 创建Client对象
      try {
        boolean doRun = client.init(args);  //2 初始化
        if (!doRun) {
          System.exit(0);
        }
      }
      result = client.run();   //3 运行
    }
    if (result) {
      System.exit(0);
    }
    System.exit(2);
  }
```

### 1 创建Client对象

创建时会指定本Client要用到的AM。 创建yarnClient。yarn将client与RM的交互抽象出了编程库YarnClient，用以应用程序提交、状态查询和控制等，简化应用程序。

```java
  public Client(Configuration conf) throws Exception  {
    this(		//指定AM
      "org.apache.hadoop.yarn.applications.distributedshell.ApplicationMaster",
      conf);
  Client(String appMasterMainClass, Configuration conf) {
    this.conf = conf;
    this.appMasterMainClass = appMasterMainClass;
    yarnClient = YarnClient.createYarnClient();		//创建yarnClient
    yarnClient.init(conf);
    opts = new Options();	//创建opts，后面解析参数的时候用
    opts.addOption("appname", true, "Application Name. Default value - DistributedShell");
    opts.addOption("priority", true, "Application Priority. Default 0");
}
```

### 2 初始化

init会解析命令行传入的参数，例如使用的jar包、内存大小、cpu个数等。 代码里使用GnuParser解析：init时定义所有的参数opts（可以认为是一个模板），
然后将opts和实际的args传入解析后得到一个CommnadLine对象，后面查询选项直接操作该CommnadLine对象即可，如cliParser.hasOption("help")和cliParser.getOptionValue("jar")。

```java
 public boolean init(String[] args) throws ParseException {
    CommandLine cliParser = new GnuParser().parse(opts, args);
    amMemory = Integer.parseInt(cliParser.getOptionValue("master_memory", "10"));
    amVCores = Integer.parseInt(cliParser.getOptionValue("master_vcores", "1"));
    shellCommand = cliParser.getOptionValue("shell_command");
    appMasterJar = cliParser.getOptionValue("jar");
    ...
```

### 3 运行

先启动yarnClient，会建立跟RM的RPC连接，之后就跟调用本地方法一样。通过此yarnClient查询NM个数、NM详细信息（ID/地址/Container个数等）、Queue info（其实没用到，示例里只是打印了下调试用）。

```java

public class Client {
  public boolean run() throws IOException, YarnException {
    yarnClient.start();
    YarnClusterMetrics clusterMetrics = yarnClient.getYarnClusterMetrics();
    List<NodeReport> clusterNodeReports = yarnClient.getNodeReports(
收集提交AM所需的信息。
    YarnClientApplication app = yarnClient.createApplication();	//创建app
    GetNewApplicationResponse appResponse = app.getNewApplicationResponse();
...
    ApplicationSubmissionContext appContext = app.getApplicationSubmissionContext();
    //AM需要的本地资源，如jar包、log文件
    Map<String, LocalResource> localResources = new HashMap<String, LocalResource>();

    FileSystem fs = FileSystem.get(conf);
    addToLocalResources(fs, appMasterJar, appMasterJarPath, appId.toString(),
        localResources, null);
    ...	//添加localResource

    vargs.add(Environment.JAVA_HOME.$$() + "/bin/java");
    vargs.add("-Xmx" + amMemory + "m");
    vargs.add(appMasterMainClass);
...
    for (CharSequence str : vargs) {
      command.append(str).append(" ");	//重新组织命令行
    }
	//创建Container加载上下文，包含本地资源，环境变量，实际命令。
    ContainerLaunchContext amContainer = ContainerLaunchContext.newInstance(
      localResources, env, commands, null, null, null);

    Resource capability = Resource.newInstance(amMemory, amVCores);
    appContext.setResource(capability);		//请求使用的内存、cpu

    appContext.setAMContainerSpec(amContainer);
    appContext.setQueue(amQueue);

```

重新组织出来的commands如下：

$JAVA_HOME/bin/java -Xmx10m org.apache.hadoop.yarn.applications.distributedshell.ApplicationMaster --container_memory 10
提交AM（即appContext），并启动监控。 Client只关心自己提交到RM的AM是否正常运行，而AM内部的多个task，由AM管理。如果Client要查询应用程序的任务信息，需要自己设计与AM的交互。
    yarnClient.submitApplication(appContext);   //客户端提交AM到RM
    return monitorApplication(appId);
总的来说，Client做的事情比较简单，即建立与RM的连接，提交AM，监控AM运行状态。

有个疑问，走读代码没有看到jar包是怎么送到NM上去的。


## Application Master解析

AM简化框架如下：

```java


      boolean doRun = appMaster.init(args);
      if (!doRun) {
        System.exit(0);
      }
      appMaster.run();
      result = appMaster.finish();
// yarn抽象了两个编程库，AMRMClient和NMClient(AM和RM都可以用)，简化AM编程。

// 1 设置RM、NM消息的异步处理方法
    AMRMClientAsync.CallbackHandler allocListener = new RMCallbackHandler();
    amRMClient = AMRMClientAsync.createAMRMClientAsync(1000, allocListener);
    amRMClient.init(conf);
    amRMClient.start();

    containerListener = createNMCallbackHandler();
    nmClientAsync = new NMClientAsyncImpl(containerListener);
    nmClientAsync.init(conf);
    nmClientAsync.start();
// 2 向RM注册
    RegisterApplicationMasterResponse response = amRMClient.registerApplicationMaster(appMasterHostname,
        appMasterRpcPort, appMasterTrackingUrl);
// 3 计算需要的Container，向RM发起请求
    // Setup ask for containers from RM
    // Send request for containers to RM
    // Until we get our fully allocated quota, we keep on polling RM for
    // containers
    // Keep looping until all the containers are launched and shell script
    // executed on them ( regardless of success/failure).
    for (int i = 0; i < numTotalContainersToRequest; ++i) {
      ContainerRequest containerAsk = setupContainerAskForRM();
      amRMClient.addContainerRequest(containerAsk);		//请求指定个数的Container
    }

  private ContainerRequest setupContainerAskForRM() {
    Resource capability = Resource.newInstance(containerMemory,
      containerVirtualCores);		//指定需要的memory/cpu能力
    ContainerRequest request = new ContainerRequest(capability, null, null,
        pri);


4 // RM分配Container给AM，AM启动任务RMCallbackHandler RM消息的响应，由RMCallbackHandler处理。示例中主要对前两种消息进行了处理。

  private class RMCallbackHandler implements AMRMClientAsync.CallbackHandler {
    //处理消息：Container执行完毕。在RM返回的心跳应答中携带。如果心跳应答中有已完成和新分配两种Container，先处理已完成
    public void onContainersCompleted(List<ContainerStatus> completedContainers) {
...
    //处理消息：RM新分配Container。在RM返回的心跳应答中携带
    public void onContainersAllocated(List<Container> allocatedContainers) {

    public void onShutdownRequest() {done = true;}

    //节点状态变化
    public void onNodesUpdated(List<NodeReport> updatedNodes) {}

    public float getProgress() {
onContainersAllocated收到分配的Container之后，会提交任务到NM。

    public void onContainersAllocated(List<Container> allocatedContainers) {
        LaunchContainerRunnable runnableLaunchContainer =   //创建runnable容器
            new LaunchContainerRunnable(allocatedContainer, containerListener);
        Thread launchThread = new Thread(runnableLaunchContainer);	//新建线程

        // launch and start the container on a separate thread to keep
        // the main thread unblocked
        // as all containers may not be allocated at one go.
        launchThreads.add(launchThread);
        launchThread.start();	//线程中提交Container到NM，不影响主流程

//简单分析下LaunchContainerRunnable。该类实现自Runnable，其run方法准备任务命令（本例即为date）。

  private class LaunchContainerRunnable implements Runnable {
    public LaunchContainerRunnable(
        Container lcontainer, NMCallbackHandler containerListener) {
      this.container = lcontainer;		//创建时记录待使用的Container
      this.containerListener = containerListener;
    }
    public void run() {
      vargs.add(shellCommand);		//待执行的shell命令
      vargs.add(shellArgs);			//shell命令参数
      List<String> commands = new ArrayList<String>();
      commands.add(command.toString());	//转为commands

      //根据命令、环境变量、本地资源等创建Container加载上下文
      ContainerLaunchContext ctx = ContainerLaunchContext.newInstance(
              localResources, shellEnv, commands, null, allTokens.duplicate(), null);
      containerListener.addContainer(container.getId(), container);
      //异步启动Container
      nmClientAsync.startContainerAsync(container, ctx);
// onContainersCompleted的功能比较简单，收到Container执行完毕的消息，检查其执行结果，如果执行失败，则重新发起请求，直到全部完成。

// NMCallbackHandler NM消息的响应，由NMCallbackHandler处理。

//在distShell示例里，回调句柄对NM通知过来的各种事件的处理比较简单，只是修改AM维护的Container执行完成、失败的个数。这样等到有Container执行完毕后，可以重启发起请求。失败处理和上面Container执行完毕消息的处理类似，达到了上面问题里所说的loopback效果。

  static class NMCallbackHandler
    implements NMClientAsync.CallbackHandler {

    @Override
    public void onContainerStopped(ContainerId containerId) {

    @Override
    public void onContainerStatusReceived(ContainerId containerId,

    @Override
    public void onContainerStarted(ContainerId containerId,
...
总的来说，AM做的事就是向RM/NM注册回调函数，然后请求Container；得到Container后提交任务，并跟踪这些任务的执行情况，如果失败了则重新提交，直到全部任务完成。
```

#  UnmanagedAM

distShell的Client提交AM到RM后，由RM将AM分配到某一个NM上的Container，这样给AM调试带来了困难。yarn提供了一个参数，Client可以设置为Unmanaged，提交AM后，会在客户端本地起一个单独的进程来运行AM。


```java

public class UnmanagedAMLauncher {
  public void launchAM(ApplicationAttemptId attemptId)
    //创建新进程
    Process amProc = Runtime.getRuntime().exec(amCmd, envAMList.toArray(envAM));
    try {
      int exitCode = amProc.waitFor();  //等待AM进程结束
    } finally {
      amCompleted = true;
    }

  public boolean run() throws IOException, YarnException {
      appContext.setUnmanagedAM(true);		//设置为Unmanaged
      rmClient.submitApplication(appContext);	//提交AM

      ApplicationReport appReport =		//监控AM状态，如果状态变为ACCEPTED，则跳出循环，launchAM。
          monitorApplication(appId, EnumSet.of(YarnApplicationState.ACCEPTED,
            YarnApplicationState.KILLED, YarnApplicationState.FAILED,
            YarnApplicationState.FINISHED));

      if (appReport.getYarnApplicationState() == YarnApplicationState.ACCEPTED) {
        launchAM(attemptId);
```