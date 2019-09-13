---
date: 2018-05-22
title: "yarn-resourcemanager-1"
author: "邓子明"
tags:
    - hadoop
    - 源码
categories:
    - hadoop源码
comment: true
---


参考资料：

https://hortonworks.com/blog/apache-hadoop-yarn-resourcemanager/

# 提交应用程序的过程


## 1. yarnClient.submitApplication(appContext);

新建请求，最终调用： rmClient.submitApplication(request);

实际上会通过RPC调用 ClientRMService.submitApplication(SubmitApplicationRequest request)

1. 
得到APPID：ApplicationId applicationId = submissionContext.getApplicationId();

2. 
rmAppManager.submitApplication(submissionContext, System.currentTimeMillis(), user);

放到 rmAppManager 中，rmAppManager 中存放了所有的 application。
跟进去，发现调用了：

this.rmContext.getDispatcher().getEventHandler().handle(new RMAppEvent(applicationId, RMAppEventType.START));

```java
public void handle(RMAppEvent event) {
      ApplicationId appID = event.getApplicationId();
      RMApp rmApp = this.rmContext.getRMApps().get(appID);
      if (rmApp != null) {
        try {
          rmApp.handle(event);
        } catch (Throwable t) {
          LOG.error("Error in handling event type " + event.getType()
              + " for application " + appID, t);
        }
      }
    }
```

然后导致这个 applicationId 所在的 RMAppEvent 状态机发生变化。

## 2.RegisterApplicationMasterResponse response = amRMClient.registerApplicationMaster(appMasterHostname, appMasterRpcPort,appMasterTrackingUrl);

注册 ApplicationMaster，注意这段代码是在用户编写的 ApplicationMaster 类中，所以这段代码运行在yarn给APPMaster分配的Container中。

RegisterApplicationMasterResponse response = client.registerApplicationMaster(appHostName, appHostPort, appTrackingUrl);

会调用：RegisterApplicationMasterResponse response = rmClient.registerApplicationMaster(request);

最终会通过RPC调用：ApplicationMasterServeice.registerApplicationMaster(RegisterApplicationMasterRequest request)

```java
this.rmContext
        .getDispatcher()
        .getEventHandler()
        .handle(
          new RMAppAttemptRegistrationEvent(applicationAttemptId, request
            .getHost(), request.getRpcPort(), request.getTrackingUrl()));
```

这种 RMAppAttemptEventType 类型的会 通过handle进行处理：
```java
public void handle(RMAppAttemptEvent event) {
      ApplicationAttemptId appAttemptID = event.getApplicationAttemptId();
      ApplicationId appAttemptId = appAttemptID.getApplicationId();
      RMApp rmApp = this.rmContext.getRMApps().get(appAttemptId);
      if (rmApp != null) {
        RMAppAttempt rmAppAttempt = rmApp.getRMAppAttempt(appAttemptID);
        if (rmAppAttempt != null) {
          try {
            rmAppAttempt.handle(event);
          } catch (Throwable t) {
            LOG.error("Error in handling event type " + event.getType()
                + " for applicationAttempt " + appAttemptId, t);
          }
        }
      }
    }
  }
```

和上面的 RMAppEvent 一样，会进入一个状态机进行处理。

### 1.状态机相互转换细节

上面的过程细化一下：

RMAppImpl 收到 RMAppEventType.START 事件后，会调用 RMStateStore#storeApplication，以日志记录 RMAppImpl 当前信息，

至此，RMAppImpl 的运行状态由 NEW 转移为 NEW_SAVING。该步骤就较为复杂了，下面详细介绍下。

其中 RMAppEventType 注册到中央异步调度器的地方在 ResourceManager.java 中，new ApplicationEventDispatcher(rmContext) 进行处理，
处理方式很简单：通过appid得到得到 RMAppImpl ，最终会给  RMAppImpl自己处理，进入他的状态机处理。状态机有这么一个事件：

addTransition(RMAppState.NEW, RMAppState.NEW_SAVING, RMAppEventType.START, new RMAppNewlySavingTransition())

RMAppNewlySavingTransition 的 transition 就是 app.rmContext.getStateStore().storeNewApplication(app);  实际上就是保存应用的相关信息。
```java
public synchronized void storeNewApplication(RMApp app) {  
    //app=RMAppImpl  
    LOG.info("begin to storeNewApplication,app="+app.toString());  
    ApplicationSubmissionContext context = app.getApplicationSubmissionContext();  
    assert context instanceof ApplicationSubmissionContextPBImpl;  
    ApplicationState appState =  
        new ApplicationState(app.getSubmitTime(), app.getStartTime(), context,app.getUser());  
    dispatcher.getEventHandler().handle(new RMStateStoreAppEvent(appState));  
  }  
```

注意： dispatcher.getEventHandler().handle(new RMStateStoreAppEvent(appState));  这里会调用 RMStateStore 状态机的 transition，实际上就是 store + notifyDoneStoringApplication

`rmDispatcher.getEventHandler().handle(new RMAppNewSavedEvent(appId, storedException)); `

这个事件又会进入 RMAppImpl 的状态机，对应代码 addTransition(RMAppState.NEW_SAVING, RMAppState.SUBMITTED, RMAppEventType.APP_NEW_SAVED, new AddApplicationToSchedulerTransition())   

调用：app.handler.handle(new AppAddedSchedulerEvent(app.applicationId,app.submissionContext.getQueue(), app.user));  

会触发： RMAppImpl 处理 AppAddedSchedulerEvent

然后这个事件会分配给：CapacityScheduler ，
```java
case APP_ADDED:  
    {  
      AppAddedSchedulerEvent appAddedEvent = (AppAddedSchedulerEvent) event;  
      addApplication(appAddedEvent.getApplicationId(),  
        appAddedEvent.getQueue(), appAddedEvent.getUser());  
```

addApplication 会调用 rmContext.getDispatcher().getEventHandler().handle(new RMAppEvent(applicationId, RMAppEventType.APP_ACCEPTED));  

RMAppImpl 会触发 ：addTransition(RMAppState.SUBMITTED, RMAppState.ACCEPTED,  RMAppEventType.APP_ACCEPTED, new StartAppAttemptTransition())  

对应的transition： createNewAttempt(); handler.handle(new RMAppStartAttemptEvent(currentAttempt.getAppAttemptId(),  transferStateFromPreviousAttempt));  
实际上就是触发 RMAppAttemptImpl 状态机操作。

RMAppAttemptImpl 接受 RMAppAttemptEventType.START 事件后，进行一系列初始化工作。将自身状态由NEW转换为SUBMITTED，并调用 AttemptStartedTransition。

AttemptStartedTransition appAttempt.eventHandler.handle(new AppAttemptAddedSchedulerEvent(  appAttempt.applicationAttemptId, transferStateFromPreviousAttempt));  

AppAttemptAddedSchedulerEvent 会交给 CapacityScheduler 。

```java
case APP_ATTEMPT_ADDED:  
    {  
      AppAttemptAddedSchedulerEvent appAttemptAddedEvent =  
          (AppAttemptAddedSchedulerEvent) event;  
      addApplicationAttempt(appAttemptAddedEvent.getApplicationAttemptId(),  
        appAttemptAddedEvent.getTransferStateFromPreviousAttempt(),  
        appAttemptAddedEvent.getShouldNotifyAttemptAdded());  
    }  
```

实际上就是讲这个 attempt 放进队列，等待处理。并且：rmContext.getDispatcher().getEventHandler().handle( new RMAppAttemptEvent(applicationAttemptId, RMAppAttemptEventType.ATTEMPT_ADDED));  

RMAppAttemptImpl 接受到事件 RMAppAttemptEventType.ATTEMPT_ADDED 后，状态由SUBMITTED转换为SCHEDULED。进入内部类ScheduleTransition的transition函数：

```java
private static final class ScheduleTransition  
      implements  
      MultipleArcTransition<RMAppAttemptImpl, RMAppAttemptEvent, RMAppAttemptState> {  
    @Override  
    public RMAppAttemptState transition(RMAppAttemptImpl appAttempt,  
        RMAppAttemptEvent event) {  
        LOG.info("class::ScheduleTransition, func::transition, begin.");  
      if (!appAttempt.submissionContext.getUnmanagedAM()) {  
        // Request a container for the AM.  
        ResourceRequest request =  
            BuilderUtils.newResourceRequest(  
                AM_CONTAINER_PRIORITY, ResourceRequest.ANY, appAttempt  
                    .getSubmissionContext().getResource(), 1);  
  
        // SchedulerUtils.validateResourceRequests is not necessary because  
        // AM resource has been checked when submission  
        Allocation amContainerAllocation = appAttempt.scheduler.allocate(  
            appAttempt.applicationAttemptId,  
            Collections.singletonList(request), EMPTY_CONTAINER_RELEASE_LIST, null, null);  
        if (amContainerAllocation != null  
            && amContainerAllocation.getContainers() != null) {  
          assert (amContainerAllocation.getContainers().size() == 0);  
        }  
        return RMAppAttemptState.SCHEDULED;  
      } else {  
        // save state and then go to LAUNCHED state  
        appAttempt.storeAttempt();  
        return RMAppAttemptState.LAUNCHED_UNMANAGED_SAVING;  
      }  
    }  
  } 
   
```

这里面就是：新建资源 ResourceRequest ，然后 appAttempt.scheduler.allocate

------ 这里断层了,谁触发了 AMContainerImpl 启动和分配 Container，需要后续再看。

这里有个疑问需要解答一下，之前一直好奇是哪里启动了 AMContainerImpl，上面的 schedule.allocate 将需要的资源提交给 schedule ，实际上 schedule 会分配。

```java
application.updateResourceRequests(ask);
```

这一句话，

以  FairScheduler 为例，启动服务会调用 initScheduler(conf); 里面有三行代码：

```java
schedulingThread = new ContinuousSchedulingThread();
schedulingThread.setName("FairSchedulerContinuousScheduling");
schedulingThread.setDaemon(true);
```

会有守护线程调用 continuousSchedulingAttempt(); 实际上会调用：
```java
    for (NodeId nodeId : nodeIdList) {
      FSSchedulerNode node = getFSSchedulerNode(nodeId);
      try {
        if (node != null && Resources.fitsIn(minimumAllocation,
            node.getAvailableResource())) {
          attemptScheduling(node);
        }
      } catch (Throwable ex) {
        LOG.error("Error while attempting scheduling for node " + node +
            ": " + ex.toString(), ex);
      }
    }
```
这个 attemptScheduling(node); 就会创建 AMContainerImpl 实例，至于怎么创建，需要了解各个 Schedule 的内部细节。


ResourceManager 为应用程序的 AM 分配资源后，创建一个 RMContainerImpl，并向它发送一个 RMContainerEventType.START 事件。

RMContainerImpl 收到 RMContainerEventType.START 事件后，直接向 RMAppAttemptImpl 发送一个 RMAppAttemptEventType.CONTAINER_ALLOCATED

RMAppAttemptImpl 收到 RMAppAttemptEventType.CONTAINER_ALLOCATED 事件后：调用 AMContainerAllocatedTransition：

transition函数中，调用 scheduler.allocate 获取分配的资源，scheduler 返回资源之前，会向 RMContainerImpl 发送 RMContainerEventType.ALLOCATED事件。

RMAppAttemptImpl 收到资源后，向 RMStateStore 发送 MStateStoreEventType.STORE_APP_ATTEMPT 事件请求记录日志。

至此，RMAppAttemptImpl 状态从 SCHEDULED 转换为 ALLOCATED_SAVING。

日志记录完成后，RMStateStore 向 RMAppAttemptImpl 发送 RMAppAttemptEventType.ATTEMPT_NEW_SAVED 事件。

RMAppAttemptImpl 收到 RMAppAttemptEventType.ATTEMPT_NEW_SAVED 事件后，
向 ApplicationMasterLauncher 发送 AMLauncherEventType.LAUNCH 事件，
至此，RMAppAttemptImpl 状态从 ALLOCATED_SAVING 转换为 ALLOCATED。

后面的和这里类似，不过涉及到了 RMContainer状态机，先跳过。


## 3.总结

通过这个实例我们大概了解了yarn中的RPC、调度器、服务、状态机配合的过程。
一般是客户端（可以使用户的client、nodeManager进程或者它启动的container进程）发送请求，中间通过RPC调用了ResourceManager中的某个服务，这个服务会触发一定的事件，并且返回。

例如客户端提交一个应用程序，首先有个 appid，每个appid对应的有一个 RMApp ，放在 rmAppManager 的一个map中。这个 RMApp 是一个状态机。

然后会调用 this.rmContext.getDispatcher().getEventHandler().handle(new RMAppEvent(applicationId, RMAppEventType.START));

调度器会启动对应的 EventHandle 去处理这个事件，而 对应的 EventHandle 会根据appid 通过 rmAppManager 得到对应的 RMApp，调用对应的状态转化函数就实现了状态转化。

再例如某个container启动 APPMaster，也是调用 
this.rmContext.getDispatcher().getEventHandler().handle
(new RMAppAttemptRegistrationEvent(applicationAttemptId, request.getHost(), request.getRpcPort(), request.getTrackingUrl()));

然后调度器会启动对应的 EventHandle 去处理这个事件，而 对应的 EventHandle 会根据appid 通过 rmAppManager 得到对应的 RMApp，
这时候事件类似是 RMAppAttemptEvent，处理逻辑变了，会在另一个状态机进行操作。

## 4.RMContainer状态机

上面分析了 两个状态机，实际上还有一个 RMContainer ，这个和上面两个类似吧。