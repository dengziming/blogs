---
date: 2019-02-22
title: "cassandra使用"
author: "邓子明"
tags:
    - cassandra
categories:
    - 开发经验
comment: true
---

## 常用命令

查看压缩转态：

bin/nodetool compactionstats 

bin/nodetool disableautocompaction 可以禁用 compact，bin/nodetool enableautocompaction 启用



## 参数配置

使用时候注意参数设置

alter table titan_20190104.edgestore with compaction = {'class': 'org.apache.cassandra.db.compaction.LeveledCompactionStrategy', 'sstable_size_in_mb': 512};
alter table titan_20190104.edgestore with compression = {'chunk_length_in_kb': '1024', 'class': 'org.apache.cassandra.io.compress.LZ4Compressor'};
alter table titan_20190104.edgestore with gc_grace_seconds = 259200;
