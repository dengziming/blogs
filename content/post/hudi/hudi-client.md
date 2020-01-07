---
date: 2019-12-30
title: "hudi-client"
author: "邓子明"
tags:
    - hudi
    - 源码
categories:
    - hudi源码
comment: true
---




## HoodieClientExample


new HoodieWriteClient

```java
// skip
sparkContext()
new Index
startEmbeddedServerView()
new HoodieCleanClient()
{
    startEmbeddedServerView();
}
```


client.startCommit();

```java
// ..

HoodieActiveTimeline.createNewInstantTime();
startCommit(commitTime);
{
    HoodieTableMetaClient metaClient = createMetaClient(true);
    {
        new HoodieTableMetaClient()
        {
            getActiveTimeline
        }
    }
    HoodieTable<T> table = HoodieTable.getHoodieTable(metaClient, config, jsc);
    HoodieActiveTimeline activeTimeline = table.getActiveTimeline();
    String commitActionType = table.getMetaClient().getCommitActionType();
    activeTimeline.createNewInstant(new HoodieInstant(State.REQUESTED, commitActionType, instantTime));
}
```

client.upsert(writeRecords, newCommitTime);

```java
HoodieTable<T> table = getTableAndInitCtx(OperationType.UPSERT);
{
    HoodieTableMetaClient metaClient = createMetaClient(true);
}
HoodieTable table = HoodieTable.getHoodieTable(metaClient, config, jsc);
JavaRDD<HoodieRecord<T>> dedupedRecords =
          combineOnCondition(config.shouldCombineBeforeUpsert(), records, config.getUpsertShuffleParallelism());
JavaRDD<HoodieRecord<T>> taggedRecords = index.tagLocation(dedupedRecords, jsc, table);
upsertRecordsInternal(taggedRecords, commitTime, table, true);
{
    JavaRDD<HoodieRecord<T>> partitionedRecords = partition(preppedRecords, partitioner);
    JavaRDD<WriteStatus> writeStatusRDD = partitionedRecords.mapPartitionsWithIndex((partition, recordItr) -> {
      if (isUpsert) {
        return hoodieTable.handleUpsertPartition(commitTime, partition, recordItr, partitioner);
      } else {
        return hoodieTable.handleInsertPartition(commitTime, partition, recordItr, partitioner);
      }
    }, true).flatMap(List::iterator);
    updateIndexAndCommitIfNeeded(writeStatusRDD, hoodieTable, commitTime);
    {
        JavaRDD<WriteStatus> statuses = index.updateLocation(writeStatusRDD, jsc, table);
        commitOnAutoCommit(commitTime, statuses, table.getMetaClient().getCommitActionType());
        {
            commit(commitTime, resultRDD, Option.empty(), actionType);
            {
                HoodieTable<T> table = HoodieTable.getHoodieTable(createMetaClient(true), config, jsc);
                HoodieActiveTimeline activeTimeline = table.getActiveTimeline();
    			HoodieCommitMetadata metadata = new HoodieCommitMetadata();
    			List<HoodieWriteStat> stats = writeStatuses.map(WriteStatus::getStat).collect();
				updateMetadataAndRollingStats(actionType, metadata, stats);
				finalizeWrite(table, commitTime, stats);
				metadata.addMetadata(HoodieCommitMetadata.SCHEMA_KEY, config.getSchema());
				activeTimeline.saveAsComplete(new HoodieInstant(true, actionType, commitTime),
			          Option.of(metadata.toJsonString().getBytes(StandardCharsets.UTF_8)));
			    // Save was a success & Do a inline compaction if enabled
			    if (config.isInlineCompaction()) {
			      metadata.addMetadata(HoodieCompactionConfig.INLINE_COMPACT_PROP, "true");
			      forceCompact(extraMetadata);
			    } else {
			      metadata.addMetadata(HoodieCompactionConfig.INLINE_COMPACT_PROP, "false");
			    }
				// We cannot have unbounded commit files. Archive commits if we have to archive
			    HoodieCommitArchiveLog archiveLog = new HoodieCommitArchiveLog(config, createMetaClient(true));
			    archiveLog.archiveIfRequired(jsc);
			    if (config.isAutoClean()) {
			      // Call clean to cleanup if there is anything to cleanup after the commit,
			      LOG.info("Auto cleaning is enabled. Running cleaner now");
			      clean(commitTime);
			    } else {
			      LOG.info("Auto cleaning is not enabled. Not running cleaner now");
			    }
			    
            }
        }
    }
}
```


