---
layout: post
title: "【译】Java垃圾回收算法[截止到Java 9]"
author: "BJ大鹏"
header-style: text
tags:
  - Java
  - 垃圾回收
---

原文：[Java Garbage Collection Algorithms [till Java 9]](https://howtodoinjava.com/java/garbage-collection/all-garbage-collection-algorithms/)

垃圾回收(Garbage collection，GC)一直是 Java 流行背后的重要特性之一。垃圾回收是 Java 中用于释放未使用内存的机制。本质上，它跟踪所有仍在使用的对象，并将其余的标记为垃圾。Java 的垃圾收集被认为是一种自动内存管理模式，因为程序员不必将对象指定为准备释放的对象。垃圾回收在低优先级线程上运行。

目录：
- [对象生命周期（Object Life Cycle）](#对象生命周期object-life-cycle)
- [垃圾回收算法（Garbage collection algorithms）](#垃圾回收算法garbage-collection-algorithms)
    - [标记清除（Mark and sweep）](#标记清除mark-and-sweep)
    - [并发标记清除垃圾回收（Concurrent mark sweep (CMS) garbage collection）](#并发标记清除垃圾回收concurrent-mark-sweep-cms-garbage-collection)
    - [串行收集器（Serial garbage collection）](#串行收集器serial-garbage-collection)
    - [并行收集器（Parallel garbage collection）](#并行收集器parallel-garbage-collection)
    - [G1(垃圾优先)收集器（G1 garbage collection）](#g1垃圾优先收集器g1-garbage-collection)
- [GC自定义选项](#gc自定义选项)
    - [GC配置标记](#gc配置标记)
    - [GC日志标记](#gc日志标记)
- [总结](#总结)

# 对象生命周期（Object Life Cycle）

Java 的对象生命周期可以分为3个阶段:
1. 对象创建（Object creation），如使用关键字new。
2. 使用中的对象（Object in use）
3. 对象销毁（Object destruction）

# 垃圾回收算法（Garbage collection algorithms）

## 标记清除（Mark and sweep）

它是初始的和非常基本的算法，分两个阶段运行:
1. 标记存活对象（Marking live objects）– 找出所有仍然活着的对象
2. 删除不可到达的对象（Removing unreachable objects）

首先，GC 将一些特定的对象定义为垃圾收集根（Garbage Collection Roots）。 例如，当前执行方法的局部变量和输入参数、活动线程、加载类的静态字段和 JNI 引用。现在，GC 遍历内存中的整个对象图，从这些根开始，然后从根引用到其他对象。 GC访问的每个对象都被标记为活动的。

第二阶段是清除未使用的对象以释放内存。这可以通过多种方式来实现，例如:
- 常规删除（Normal deletion）- 常规删除会删除未引用的对象以释放空间，并留下引用的对象和指针。 内存分配器(类似于散列表)保存对可以分配新对象的空闲空间块的引用。 它通常被称为标记-清除(mark-sweep)算法。 
<img src="https://howtodoinjava.com/wp-content/uploads/2018/04/Normal-Deletion-Mark-and-Sweep.png" /><br>
- 删除-压缩（Deletion with compacting）- 只删除未使用的对象是没有效率的，因为空闲内存块分散在存储区域中，如果创建的对象足够大并且没有找到足够大的内存块，就会导致 OutOfMemoryError 错误。 为了解决这个问题，在删除未引用的对象之后，将对剩余的引用对象进行压缩。 这里的压缩指的是将被引用的对象移动到一起的过程。 这使得新的内存分配更加容易和快速。它通常被称为标记-清除-压缩（mark-sweep-compact）算法。
<img src="https://howtodoinjava.com/wp-content/uploads/2018/04/Deletion-with-compacting.png" /><br>
- 复制删除（Deletion with copying）- 与标记和压缩的方法非常类似，因为他们也重新安置所有存活对象。 重要的区别在于迁移的目标是不同的内存区域。它通常被称为标记-复制（mark-copy）算法。
<img src="https://howtodoinjava.com/wp-content/uploads/2018/04/Deletion-with-copying-Mark-and-Sweep.png" />

## 并发标记清除垃圾回收（Concurrent mark sweep (CMS) garbage collection）

它试图通过与应用程序线程并发执行大部分垃圾收集工作来最小化垃圾收集引起的暂停。 该算法在年轻代中采用并行stop-the-world标记复制(mark-copy)算法，在老年代中采用大多并发的标记清除(mark-sweep)算法。

由于从JDK9开始CMS被标记为废弃，并且在JDK14中完全移除，因此不再介绍该收集器。

## 串行收集器（Serial garbage collection）

该算法对年轻代使用标记-复制(mark-copy)，对老年代使用标记-清除-压缩(mark-sweep-compact)。它在单线程上工作。在执行时，它会冻结所有其他线程，直到垃圾回收操作结束。

由于串行垃圾收集具有线程冻结(thread-freezing)的特性，因此只适用于非常小的程序。

要使用 Serial GC，请使用以下 JVM 参数:
```
-XX:+UseSerialGC
```

## 并行收集器（Parallel garbage collection）

类似于串行GC，年轻代使用标记-复制(mark-copy)，老年代使用标记-清除-压缩(mark-sweep-compact)。多个并发线程用于标记和复制/压缩阶段。可以使用 ```-XX:ParallelGCThreads=N``` 选项配置线程数。

并行垃圾收集器适用于主要目标是通过有效利用现有系统资源来提高吞吐量的多核机器上。使用这种方法，可以大大减少GC的周期时间。

直到Java 8，并行收集器是默认的垃圾收集器。从 Java 9开始，G1是32位和64位服务器配置的默认垃圾收集器。

要使用并行 GC，请使用以下 JVM 参数:
```
-XX:+UseParallelGC
```
## G1(垃圾优先)收集器（G1 garbage collection）

G1(Garbage First)垃圾收集器在Java 7中可用，被设计为CMS收集器的长期替代品。 G1收集器是一个并行、并发和增量压缩的低暂停垃圾收集器。

这种方法包括将内存堆分割成多个小区域(通常是2048)。每个地区被标记为年轻代(进一步分为伊甸园区域或幸存者区域)或老年代。这使得GC可以避免立即回收整个堆，而是以增量方式处理问题。这意味着一次只考虑区域的一个子集。

<img src="https://howtodoinjava.com/wp-content/uploads/2018/04/Memory-regions-marked-G1.png" />

G1持续跟踪每个区域包含的存活数据量。此信息用于确定包含最多垃圾的区域; 因此它们首先被回收。这就是为什么它被命名为垃圾优先收集。

与其他算法一样，不幸的是，压缩操作使用 Stop the World 方法进行。但是根据它的设计目标，你可以为它设定具体的性能目标。您可以配置暂停时间，例如，在任何给定的时间内，暂停时间不超过10毫秒。G1收集器将尽最大努力以高概率实现这一目标(但不是确定性的，由于操作系统级别的线程管理，这将是实时性的困难)。

如果你想在 Java 7 或 Java 8 机器上使用，请使用如下的 JVM 参数:
```
-XX:+UseG1GC
```

### G1自定义选项

<style>
table th:first-of-type {
    width: 50%;
}
</style>

|标记|描述|
|---|---|
|-XX:G1HeapRegionSize=16m|堆区域的大小。该值的大小为2的幂，范围从1MB到32MB。我们的目标是根据最小的Java 堆大小设置大约2048个区域|
|-XX:MaxGCPauseMillis=200|为期望的最大暂停时间设置目标值。默认值是200毫秒。指定的值不适合堆大小|
|-XX:G1ReservePercent=5|这决定了堆中的最小储备|
|-XX:G1ConfidencePercent=75| 这是信心系数暂停预测启发式 |
|-XX:GCPauseIntervalMillis=200|这是每个MMU的暂停间隔时间片(以毫秒为单位)|

# GC自定义选项

## GC配置标记

|标记|描述|
|---|---|
|-Xms2048m -Xmx3g|设置初始和最大堆大小（年轻代+老年代）|
|-XX:+DisableExplicitGC|这将导致JVM忽略应用程序对```System.gc()```方法的任何调用。|
|-XX:+UseGCOverheadLimit|This is the use policy used to limit the time spent in garbage collection before an OutOfMemory error is thrown.|
|-XX:GCTimeLimit=95|This limits the proportion of time spent in garbage collection before an OutOfMemory error is thrown. This is used with GCHeapFreeLimit.|
|-XX:GCHeapFreeLimit=5|This sets the minimum percentage of free space after a full garbage collection before an OutOfMemory error is thrown. This is used with GCTimeLimit.|
|-XX:InitialHeapSize=3g|设置初始堆空间（年轻区+老年区）|
|-XX:MaxHeapSize=3g|设置最大堆空间（年轻区+老年区）|
|-XX:NewSize=128m|设置年轻区的初始空间|
|-XX:MaxNewSize=128m|设置年轻区的最大空间|
|-XX:SurvivorRatio=15|设置单个幸存者空间的大小为伊甸园空间大小的一部分|
|-XX:PermSize=512m|设置永久区的初始空间|
|-XX:MaxPermSize=512m|设置永久区的最大空间|
|-Xss512k|设置专用于每个线程的栈区域的大小(以字节为单位)|

## GC日志标记

|标记|描述|
|---|---|
|-verbose:gc or -XX:+PrintGC|This prints the basic garbage collection information.|
|-XX:+PrintGCDetails|This will print more detailed garbage collection information.|
|-XX:+PrintGCTimeStamps|You can print timestamps for each garbage collection event. The seconds are sequential and begin from the JVM start time.|
|-XX:+PrintGCDateStamps|You can print date stamps for each garbage collection event.|
|-Xloggc:|Using this you can redirect garbage collection output to a file instead of the console.|
|-XX:+Print\TenuringDistribution|You can print detailed information regarding young space following each collection cycle.|
|-XX:+PrintTLAB|You can use this flag to print TLAB allocation statistics.|
|-XX:+PrintReferenceGC|Using this flag, you can print the times for reference processing (that is, weak, soft, and so on) during stop-the-world pauses.|
|-XX:+HeapDump\OnOutOfMemoryError|This creates a heap dump file in an out-of-memory condition.|

# 总结
- 对象生命周期分成3个阶段，对象创建，对象使用和对象销毁。
- 标记-清除、标记-清除-压缩、标记-复制机制如何工作。
- 不同的单线程和并发垃圾回收算法。
- 直到Java 8, 并发收集器是默认算法。
- 从Java 9开始, G1是默认算法。
- 此外，还有各种标志，用于控制垃圾收集算法的行为并记录任何应用程序的有用信息

