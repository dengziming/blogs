---
date: 2019-12-03
title: "kafka-controller"
author: "邓子明"
tags:
    - kafka
    - 源码
categories:
    - kafka源码
comment: true
---


# 分析几个操作的过程

## delete topic

first add topic to be deleted to zk path /admin/delete_topics

DeleteTopicsListener will listen to it, and then add to deleteTopicManager.

deleteTopicManager has a inner thread responsible for delete topic, which will delete the topic cooperate with replicaStateMachine,
which will cooperate with partitionStateMachine.

steps of delete topi tThread ，：
1. if isTopicEligibleForDeletion, call onTopicDeletion of deleteTopicManager，
    1.1 just get all partitions 
    1.2 send updateMetadataRequest to all living brokers to stop serving data for topicToBeDeleted
    1.3 just call onPartitionDeletion in cycle
        1.3.1 get all replicasPerPartition
        1.3.2 call startReplicaDeletion in cycle
            1.3.2.1 replicaStateMachine.handleStateChanges(deadReplicasForTopic, ReplicaDeletionIneligible)
            1.3.2.2 replicaStateMachine.handleStateChanges(replicasForDeletionRetry, OfflineReplica)
            1.3.2.3 replicaStateMachine.handleStateChanges(replicasForDeletionRetry, ReplicaDeletionStarted, new Callbacks.CallbackBuilder().stopReplicaCallback(deleteTopicStopReplicaCallback).build)
                1.3.2.3.1 if failed, controller.replicaStateMachine.handleStateChanges(replicasThatFailedToDelete, ReplicaDeletionIneligible), markTopicIneligibleForDeletion(topics), resumeTopicDeletionThread()
                1.3.2.3.2 if succeed, controller.replicaStateMachine.handleStateChanges(successfullyDeletedReplicas, ReplicaDeletionSuccessful),  resumeTopicDeletionThread()
            1.3.2.4 if deadReplicasForTopic, markTopicIneligibleForDeletion(Set(topic))
2. if areAllReplicasForTopicDeleted, call completeDeleteTopic
    2.1 
3. if any ReplicaDeletionIneligible, call markTopicForDeletionRetry