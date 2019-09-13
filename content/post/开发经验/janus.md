---
date: 2018-03-22
title: "常用sql写法"
author: "邓子明"
tags:
    - janusgraph
categories:
    - 开发经验
comment: true
---

## sql语法

2）删库前，执行/sdc/node1/bin/nodetool compactionstats  看下当前有没跟要删除相关的Compaction任务，如果有就执行 /sdc/node1/bin/nodetool stop命令中止,  这个是每个节点都要确认.
3)  drop keyspace后，最好删除磁盘上的物理目录, 防止复用时有影响.