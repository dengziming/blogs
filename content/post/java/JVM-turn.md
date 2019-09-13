---
date: 2018-10-24
title: "JVM调优"
author: "邓子明"
tags:
    - java
    - janusgraph
categories:
    - java
comment: true
---


## problem

janusgraph 导数据工具，数据量大的时候，一直卡主。调参意义不大。
通过 -XX:-UseGCOverheadLimit -verbose:gc -XX:+PrintGCDetails  并不能看出啥信息。


## fix

### jmap

```shell
[yangzhiyong@d28-235 graph_kg]$ jmap -heap 15294
Attaching to process ID 15294, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.74-b02

using thread-local object allocation.
Parallel GC with 23 thread(s)

Heap Configuration:
   MinHeapFreeRatio         = 0
   MaxHeapFreeRatio         = 100
   MaxHeapSize              = 32210157568 (30718.0MB)
   NewSize                  = 715653120 (682.5MB)
   MaxNewSize               = 10736369664 (10239.0MB)
   OldSize                  = 1431830528 (1365.5MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
PS Young Generation
Eden Space:
   capacity = 5928648704 (5654.0MB)
   used     = 5928648704 (5654.0MB)
   free     = 0 (0.0MB)
   100.0% used
From Space:
   capacity = 2357723136 (2248.5MB)
   used     = 0 (0.0MB)
   free     = 2357723136 (2248.5MB)
   0.0% used
To Space:
   capacity = 2357723136 (2248.5MB)
   used     = 0 (0.0MB)
   free     = 2357723136 (2248.5MB)
   0.0% used
PS Old Generation
   capacity = 21473787904 (20479.0MB)
   used     = 21473521856 (20478.74627685547MB)
   free     = 266048 (0.25372314453125MB)
   99.99876105696308% used

Exception in thread "main" java.lang.reflect.InvocationTargetException
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at sun.tools.jmap.JMap.runTool(JMap.java:201)
	at sun.tools.jmap.JMap.main(JMap.java:130)
Caused by: sun.jvm.hotspot.oops.UnknownOopException
	at sun.jvm.hotspot.oops.ObjectHeap.newOop(ObjectHeap.java:263)
	at sun.jvm.hotspot.memory.StringTable.stringsDo(StringTable.java:72)
	at sun.jvm.hotspot.tools.HeapSummary.printInternStringStatistics(HeapSummary.java:299)
	at sun.jvm.hotspot.tools.HeapSummary.run(HeapSummary.java:148)
	at sun.jvm.hotspot.tools.Tool.startInternal(Tool.java:260)
	at sun.jvm.hotspot.tools.Tool.start(Tool.java:223)
	at sun.jvm.hotspot.tools.Tool.execute(Tool.java:118)
	at sun.jvm.hotspot.tools.HeapSummary.main(HeapSummary.java:49)
	... 6 more
[yangzhiyong@d28-235 graph_kg]$
```

jmap 可以看出： 配置的堆内存是3G左右，Eden Space  和 PS Old Generation  已经用完了，From 和 TO 还没用， 有 UnknownOopException，问题看起来很复杂。

## jstack 


jstack -F 15294



##

原因:  java.lang.OutOfMemoryError: GC overhead limit exceeded

Cassandra 的 sstableloader 有这个问题