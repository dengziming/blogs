---
date: 2018-05-22
title: "yarn-基础库"
author: "邓子明"
tags:
    - hadoop
    - 源码
categories:
    - hadoop源码
comment: true
---


# yarn-事件库和服务库

## 使用
1. 新建Event和EventType
2. 新建 AsyncDispatcher 并给 AsyncDispatcher 注册 Event 和对应的 EventHandler<Event>
3. 调用 AsyncDispatcher 的 getEventHandler 得到 EventHandler 然后调用 handler 的 handle 方法处理 Event

## 基本原理：

AsyncDispatcher 注册 EventHandler<Event> 的过程实际上生成了一个 map，保存了每个事件对应的handler。同时有一个 队列，用于放置 Event

调用 handle 的时候 将Event放进queue中，内部启动一个线程不断处理 queue的任务。

# yarn-状态机

## 使用

1. 初始化
```java
StateMachineFactory
.addTransition(JobStateInternal.NEW, JobStateInternal.INITED, JobEventType.JOB_INIT,new InitTransition())
.addTransition(JobStateInternal.INITED, JobStateInternal.SETUP, JobEventType.JOB_START,new StartTransition())
.installTopology()
.make()
```

2. 新建对应的 Transition

```java
public static class InitTransition implements SingleArcTransition<JobStateMachine,JobEvent>{

        @Override
        public void transition(JobStateMachine job, JobEvent event) {
            System.out.println("Receiving event " + event);
        }

    }
```

3. 调用 StateMachine 的 doTransition(event.getType(), event)

## 原理

installTopology的时候创建一个拓扑图，记录每个 State 能接受的 Event，以及接受该 Event 后的操作，以及操作后的 State。

每次有Event传入，调用对应的 Transition ，并且将 此时刻 的状态变为 操作后的状态。

