---
title: 垃圾回收器之CMS
date: 2018-11-17 18:19:18
tags:
    -  JVM
    -  GC
---
JVM CMS垃圾回收器探索
<!--more-->

#### CMS简介

&emsp;CMS垃圾回收器是针对老年代的回收方式、是以牺牲吞吐量为代价来获得最短回收停顿时间的垃圾回收器。对于要求服务器响应速度的应用上，这个方式非常适合。在JVM启动参数上面加上 `XX:+UseConcMarkSweepGC` 表示老年代使用CMS的方式回收，CMS的回收算法是&ensp;**标记-清除**

#### CMS执行过程

##### &emsp;STW initial mark
```
2018-11-21T11:41:46.066+0800: 58424.191: [GC (CMS Initial Mark) [1 CMS-initial-mark: 3636058K(5592448K)] 3691980K(7829376K), 0.0037087 secs] [Times: user=0.06 sys=0.00, real=0.00secs]
```
&emsp;**初始标记：** 这个阶段JVM停止正在执行的任务。从根对象开始扫描和root对象直接关联的对象。这个过程会很快。

##### &emsp;Concurrent marking 
```
2018-11-21T11:41:46.070+0800: 58424.195: [CMS-concurrent-mark-start]
2018-11-21T11:41:46.122+0800: 58424.247: [CMS-concurrent-mark: 0.051/0.051 secs] [Times: user=0.57 sys=0.00, real=0.05 secs]
```
&emsp;**并发标记：** 这个阶段JVM标记线程和用户线程并发执行。在初始标记的基础上继续向下追溯扫描对象，这个过程不会停顿。

##### &emsp;Concurrent precleaning
```
2018-11-21T11:41:46.122+0800: 58424.247: [CMS-concurrent-preclean-start]
2018-11-21T11:41:46.143+0800: 58424.268: [CMS-concurrent-preclean: 0.021/0.021 secs] [Times: user=0.04 sys=0.01, real=0.02 secs]
```
&emsp;**并发预清理：** 这个阶段JVM标记线程和用户线程并发执行。JVM查找在并发标记阶段新进入老年代的对象。通过从新扫描减少下一个阶段的工作，这个过程不会停顿。

##### &emsp;STW remark 
```
2018-11-21T11:41:47.486+0800: 58425.611: [GC (CMS Final Remark) [YG occupancy: 1078027 K (2236928 K)]
2018-11-21T11:41:47.486+0800: 58425.611: [GC (CMS Final Remark) 
2018-11-21T11:41:47.486+0800: 58425.611: [ParNew: 1078027K->40803K(2236928K), 0.0066178 secs] 4714086K->3676908K(7829376K), 0.0069331 secs] [Times: user=0.14 sys=0.00, real=0.00 secs]
2018-11-21T11:41:47.493+0800: 58425.618: [Rescan (non-parallel) 
2018-11-21T11:41:47.493+0800: 58425.618: [grey object rescan, 0.0209332 secs]
2018-11-21T11:41:47.514+0800: 58425.639: [root rescan, 0.0128484 secs]
2018-11-21T11:41:47.527+0800: 58425.652: [visit unhandled CLDs, 0.0000249 secs]
2018-11-21T11:41:47.527+0800: 58425.652: [dirty klass scan, 0.0011690 secs], 0.0350621 secs]
2018-11-21T11:41:47.528+0800: 58425.653: [weak refs processing, 0.0536894 secs]
2018-11-21T11:41:47.582+0800: 58425.707: [class unloading, 0.0251479 secs]
2018-11-21T11:41:47.607+0800: 58425.732: [scrub symbol table, 0.0052907 secs]
2018-11-21T11:41:47.613+0800: 58425.737: [scrub string table, 0.0009897 secs][1 CMS-remark: 3636104K(5592448K)] 3676908K(7829376K), 0.1292316 secs] [Times: user=0.25 sys=0.01, real=0.13 secs]
```
&emsp;**重新标记：** 这个阶段JVM停止正在执行的任务。最终确认老年代中存活的对象，因为之前的处理都是并发的，应用程序也在不停的分配对象。

##### &emsp;Concurrent sweeping
```
2018-11-21T11:41:47.616+0800: 58425.741: [CMS-concurrent-sweep-start]
2018-11-21T11:41:50.523+0800: 58428.648: [CMS-concurrent-sweep: 2.889/2.908 secs] [Times: user=5.07 sys=0.13, real=2.91 secs]
```
&emsp;**并发清理：** 这个阶段JVM清理线程和用户线程并发执行，清理死亡对象。

##### &emsp;Concurrent reset
```
2018-11-21T11:41:50.524+0800: 58428.649: [CMS-concurrent-reset-start]
2018-11-21T11:41:50.537+0800: 58428.662: [CMS-concurrent-reset: 0.014/0.014 secs] [Times: user=0.03 sys=0.00, real=0.01 secs]
```
&emsp;**并发重置：** 重置CMS收集器数据结构，等待下一次回收。

#### CMS参数优化
1.  由于CMS垃圾回收器使用的是标记-清除回收算法，随着GC的执行老年代会逐渐产生内存碎片,可以使用 `-XX:+UseCMSCompactAtFullCollection` 这个参数在每次执行CMS GC之后进行一次压缩处理。可以根据自己的应用老年代对象的情况调整 `-XX:CMSFullGCsBeforeCompaction=1` 这个参数来控制每隔几次GC之后进行压缩操作。
2.  CMS在垃圾回收的过程当中，会不停的有新对象在老年代分配，在CMS回收之前必须留有一部分空间，所以CMS不会在老年代满了的时候开始回收，使用 `-XX:CMSInitiatingOccupancyFraction=65 -XX:+UseCMSInitiatingOccupancyOnly` 这个两个参数来配置老年代空间到达什么阈值之后开始回收。特别注意的是如果不配置第二个参数，CMS只会在第一次回收的按照设置的阈值，后面CMS会根据情况动态调整阈值参数。
3.  由CMS回收的过程中我们可以知道，在初始标记和重新标记两个阶段会STW，所以为了使得这两个阶段尽可能快一点执行完成，可以使用 `-XX:+CMSParallelInitialMarkEnabled -XX:+CMSParallelRemarkEnabled` 这个参数来开启并行标记。
4.  在CMS回收的过程中，耗时最长的是重新标记阶段，可以使用 `-XX:+CMSScavengeBeforeRemark` 这个参数在重新标记之前执行一次Young GC。这样Young区待标记的对象就会减少很多，重新标记阶段的工作量就会少很多。因为执行Young GC也有一定的耗时，所以这个参数需要自己做一个权衡。
