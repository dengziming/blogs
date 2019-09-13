---
date: 2018-03-22
title: "janus 常用语句总结"
author: "邓子明"
tags:
    - 开发经验
    - janusgraph
categories:
    - janusgraph
comment: true
---


## 查询一批顶点周围所有的节点。

graph = JanusGraphFactory.open('conf/online/online_u2u_20181129.properties');
g = graph.traversal();
vertIds=[824746000,41257885816,41418600552,41436626944,41687556288,82284519496,287094591600]
g.V(vertIds).aggregate('a').bothE().where(inV().where(bothE('a'))).inV()


graph = JanusGraphFactory.open('conf/online/online_titan.properties');
g = graph.traversal();


## 网络拓展

