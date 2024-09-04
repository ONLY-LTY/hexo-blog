---
title: CMS Failed
date: 2018-11-17 16:40:00
tags:
    - JVM
    - GC
---
CMS 日志Promotion Failed 和 Concurrent Model Failed 错误分析

##### Promotion Failed

```java
内存情况
Heap Configuration:
   MinHeapFreeRatio         = 40
   MaxHeapFreeRatio         = 70
   MaxHeapSize              = 8589934592 (8192.0MB)
   NewSize                  = 2863267840 (2730.625MB)
   MaxNewSize               = 2863267840 (2730.625MB)
   OldSize                  = 5726666752 (5461.375MB)
   NewRatio                 = 2
   SurvivorRatio            = 3
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)
```
```java
错误日志
2018-11-15T09:01:50.794+0800: 51454.418: [GC (Allocation Failure) 2018-11-15T09:01:50.794+0800: 51454.419: [ParNew (promotion failed): 1782424K->1769555K(2236928K), 0.8356030 secs]2018-11-15T09:01:51.630+0800: 51455.254: [CMS: 3509138K->874891K(5592448K), 4.5982825 secs] 5290213K->874891K(7829376K), [Metaspace: 56929K->56929K(1101824K)], 5.4344311 secs] [Times: user=3.61 sys=2.44, real=5.43 secs]
```
上面是一次GC日志、从错误信息来看是Eden区的对象晋升失败了。从日志我们看出来Young GC的时候只回收了（178242-176955）12869k对象、剩下176955k(1.68G)存活对象需要放入到Survivor区、由上面的配置我们推算出Survivor区只有546M的大小、这个时候肯定需要放入Old区了。而此时的Old区剩余（5290213k-3509138k）1.69G的内存、需要放入（1.68G-546M）1.14G大小的对象。这里是不是觉得应该放的下。其实CMS使用的标记清理的内存回收算法，随着时间的迁移会逐渐产生内存碎片。此时如果存活对象中有大对象而且内存碎片不足以分配就会发生上面日志中的那种情况。这个时候CMS会执行GC回收Old区然后放存活的对象。由于整个过程是在ParNew阶段，所以整个过程是STW的。后果可想而知。
其实问题的核心是Young GG的时候为什么有那么多存活的对象。这个问题就要根据自己当时的具体情况去分析了。

#####  Concurrent Model Failed

```java
2018-11-21T20:22:38.029+0800: 447086.973: [GC (Allocation Failure) 2018-11-21T20:22:38.029+0800: 447086.973: [ParNew: 559232K->559232K(559232K), 0.0000426 secs]2018-11-21T20:22:38.029+0800: 447086.973: [CMS2018-11-21T20:22:38.482+0800: 447087.426: [CMS-concurrent-sweep: 0.442/0.467 secs] [Times: user=0.48 sys=0.03, real=0.47 secs]
(concurrent mode failure): 1205455K->965029K(1398144K), 2.3149339 secs] 1764687K->965029K(1957376K), [Metaspace: 54873K->54873K(1099776K)], 2.3157339 secs] [Times: user=2.15 sys=0.17, real=2.31 secs]
```

CMS GC的执行过程中，由于是并发执行，在回收的过程中会不断有新的对象分配。如果回收的速度赶不上新对象在老年代的分配速度会导致在GC的过程中老年代已经满了，这个时候会STW，执行一次老年代的回收。(这个错误会发生在CMS GC三个并发阶段的任一阶段)。遇到这个错误可以尝试调整堆空间。或者根据具体情况调整CMS GC开始回收的阈值。
