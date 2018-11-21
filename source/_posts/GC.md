---
title: Promotion Failed 
date: 2018-11-17 16:40:00
tags:
    - JVM
    - GC
---
GC 日志Promotion Failed 错误分析
<!--more-->

JVM参数配置(优化过之后的参数)

>jvm_args="-Xmx8192m -Xms8192m -verbose:gc -Xloggc:$logPATH/gc.log -XX:CMSInitiatingOccupancyFraction=65 -XX:+UseCMSInitiatingOccupancyOnly -XX:+CMSScavengeBeforeRemark -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=1 -XX:MaxTenuringThreshold=10 -XX:-UseAdaptiveSizePolicy -XX:PermSize=256M -XX:MaxPermSize=512M -XX:SurvivorRatio=3  -XX:NewRatio=2 -XX:+PrintGCDateStamps  -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:+PrintGCDetails -XX:+AlwaysPreTouch -XX:-CMSParallelRemarkEnabled"

GC错误日志

>2018-11-15T09:01:50.794+0800: 51454.418: [GC (Allocation Failure) 2018-11-15T09:01:50.794+0800: 51454.419: [ParNew (promotion failed): 1782424K->1769555K(2236928K), 0.8356030 secs]2018-11-15T09:01:51.630+0800: 51455.254: [CMS: 3509138K->874891K(5592448K), 4.5982825 secs] 5290213K->874891K(7829376K), [Metaspace: 56929K->56929K(1101824K)], 5.4344311 secs] [Times: user=3.61 sys=2.44, real=5.43 secs]

上面是一次GC日志、从错误信息来看是Eden区的对象晋升失败了。从日志我们看出来Young GC的时候只回收了（178242-176955）12869k对象、剩下176955k(1.68G)存活对象需要放入到Survivor区、由上面的配置我们推算出Survivor区只有546M的大小、这个时候肯定需要放入Old区了。而此时的Old区剩余（5290213k-3509138k）1.69G的内存、需要放入（1.68G-546M）1.14G大小的对象。这里是不是觉得应该放的下。其实CMS使用的标记清理的内存回收算法，随着时间的迁移会逐渐产生内存碎片。此时如果存活对象中有大对象而且内存碎片不足以分配就会发生上面日志中的那种情况。这个时候CMS会执行GC回收Old区然后放存活的对象。由于整个过程是在ParNew阶段，所以整个过程是STW的。后果可想而知。
其实问题的核心是Young GG的时候为什么有那么多存活的对象。这个问题就要根据自己当时的具体情况去分析了。
