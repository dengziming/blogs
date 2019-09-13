---
date: 2018-05-22
title: "yarn-nodemanager-剖析"
author: "邓子明"
tags:
    - hadoop
    - 源码
categories:
    - hadoop源码
comment: true
---


# 架构

ContainerManagementImpl 



# Container 生命周期

第一步是 RM 的 applicationMasterLauncher ，创建 ApplicationMasterLauncher 后，遇到 launch 时间 ，
case LAUNCH: launch(application); => new AMLauncher(context, application, event, getConfig());

这个任务放进 队列里面等待执行，一旦执行会调用 launch() 方法，然后调用 containerMgrProxy.startContainers(allRequests); 这是 RPC 调用

实际上就是 ContainerManagementImpl ，然后会调用 startContainerInternal ，然后就是 new ContainerImpl 。

这是 APPMaster 启动需要的 container ，实际上还有 APPMaster 调度任务需要更多的 Container ，继续向 ContainerManagementImpl 请求

## 1. 资源本地化

实际上就是 ContainerManagementImpl ，然后会调用 startContainerInternal ，然后就是 new ContainerImpl 。
然后通过 if (null == context.getApplications().putIfAbsent(applicationID,application)) 判断是否是该 NodeManager 第一个 Container ，如果是的话，new ApplicationImpl
向 ApplicationImpl 发送 ApplicationInitEvent 事件，同时发送 ApplicationContainerInitEvent 事件。

这些事件会触发 ACL、log等相关的事件， 收到 ApplicationContainerInitEvent 后将 Container 加入 ApplicationImpl 的维护列表。

logHandle 处理完成之后会发送一个 log 事件，applicationImpl 收到后向 ResourceLocalizeService 发送 事件，
为 private 和 application 级别的资源创建 LocalResourceTrackerImp ，为下载资源作准备。

private 的资源用户可见，如果该用户已经提交过了，无需创建。同理，如果 application 已经启动过 container 了，则同一个 application 的新 container 不必在创建。

经过上面操作后，ResourceLocalizeService 向 ApplicationImpl 发送 Application_Init 

ApplicationImpl 收到 INIT 后，向所有的 ContainerImpl 发送 InitContainer ，ApplicationImpl 也从 ApplicationState.INITING 变为 ApplicationState.RUNNING,

InitContainer 命令后，和 AuxService 交互，然后从 ContainerLaunchContext 得到各类可见性资源并保存到相应数据结构，然后发送给 ResourceLocalizeService 。

ResourceLocalizeService 调用 handleInitContainerResources((ContainerLocalizationRequestEvent) event); 实际是 是发送给 LocalResourcesTrackerImpl 。

LocalResourcesTrackerImpl 会 判断是否需要下载等，为对应的资源创建 LocalizedResource 状态机，将 Request 发送给 LocalizedResource。

后续还是这样的时间驱动，总之可以概括为 ： NodeManager 上同一个 App 所有的 ContainerImpl 异步并发向向资源下载服务 ResourceLocalizeService 发送待下载的资源，
ResourceLocalizeService下载完成后会通知依赖资源的所以 Container ，当一个 Container 依赖的资源全部下载完毕，Container 将会进入 运行阶段

## 2. Container 运行

运行是 ContainerLauncher 服务实现的，主要过程为： 将待运行 Container 所需要的环境变量和运行命令写到 `launch_container.sh` 中，
将启动该脚本的命令写入：`default_container_executor.sh` 中。

通过运行该脚本启动 Container 。主要有四步：

1. ContainerImpl 向 ContainersLauncher 发送 Launch_container ，请求启动 container。 
dispatcher.getEventHandler().handle(new ContainersLauncherEvent(this, launcherEvent));

2. ContainersLauncher 收到后，


```java
Application app =context.getApplications().get(containerId.getApplicationAttemptId().getApplicationId());
ContainerLaunch launch = new ContainerLaunch(context, getConfig(), dispatcher, exec, app,event.getContainer(), dirsHandler, containerManager);
containerLauncher.submit(launch);
running.put(containerId, launch);
break;
```


ContainerLaunch 放到线程池执行，对应的 call 方法为：

为 Container 创建 token 文件 和 `launch_container.sh` ，将他们保存到 NodeManager 私有目录 nmPrivate 下面， `launch_container.sh`包含了运行所以的命令。
一般都是前面 export 环境变量，最后有个 exec 命令 。

3. 准备好了 命令，
`Container_Launcher` 首先向 ContainerImpl 发送 `Container_LANUCHED` 命令，然他启动监控等。然后调用 ContainerExector launchContainer 启动 Container 。

然后是启动监控，汇报信息等。
