---
title: JVM性能调优经验总结
link_title: java-jvm-optimization
date: 2017-08-16 09:51:29
tags: [Java, JVM]
categories: Java
thumbnailImage: https://i.loli.net/2019/09/24/lrX2pLFfaCK4xVt.png	
thumbnailImagePosition: left
---
<span/>
<!-- more -->
![](https://i.loli.net/2019/09/24/lrX2pLFfaCK4xVt.png)
<!-- toc -->
# 说明
调优是一个循序渐进的过程，必然需要经历多次迭代，最终才能换取一个较好的折中方案。

在JVM调优这个领域，没有任何一种调优方案是适用于所有应用场景的，同时，切勿极端才能够达到JVM性能调优的真正目的和意义。

# 调优策略
## 核心目标
 - GC的时间足够的小
 - GC的次数足够的少
 - 发生Full GC的周期足够的长
 
 前两个目前是相悖的，要想GC时间小必须要一个更小的堆，要保证GC次数足够少，必须保证一个更大的堆，我们只能取其平衡。

 更大的年轻代必然导致更小的年老代，大的年轻代会延长普通GC的周期，但会增加每次GC的时间；小的年老代会导致更频繁的Full GC
 
 更小的年轻代必然导致更大年老代，小的年轻代会导致普通GC很频繁，但每次的GC时间会更短；大的年老代会减少Full GC的频率
 
 如何选择应该依赖应用程序对象生命周期的分布情况：如果应用存在大量的临时对象，应该选择更大的年轻代；如果存在相对较多的持久对象，年老代应该适当增大。
 
 但很多应用都没有这样明显的特性，在抉择时应该根据以下两点：a、本着Full GC尽量少的原则，让年老代尽量缓存常用对象，JVM的默认比例3：8也是这个道理（B）通过观察应用一段时间，看其他在峰值时年老代会占多少内存，在不影响FullGC的前提下，根据实际情况加大年轻代，比如可以把比例控制在1：1。但应该给年老代至少预留1/3的增长空间


## 将新对象预留在年轻代
众所周知，**由于 Full GC 的成本远远高于 Minor GC，因此某些情况下需要尽可能将对象分配在年轻代**，这在很多情况下是一个明智的选择。虽然在大部分情况下，JVM 会尝试在 Eden 区分配对象，但是由于空间紧张等问题，很可能不得不将部分年轻对象提前向年老代压缩。因此，在 JVM 参数调优时可以为应用程序分配一个合理的年轻代空间，以最大限度避免新对象直接进入年老代的情况发生。

通过设置一个较大的年轻代预留新对象，设置合理的 Survivor 区并且提供 Survivor 区的使用率，可以将年轻对象保存在年轻代。**一般来说，Survivor 区的空间不够，或者占用量达到 50%时，就会使对象进入年老代(不管它的年龄有多大)**

我们可以尝试加上
```xml
-XX:TargetSurvivorRatio=90 
```
参数，这样可以提高 from 区的利用率，使 from 区使用到 90%时，再将对象送入年老代

## 让大对象进入年老代
我们在大部分情况下都会选择将对象分配在年轻代。但是，对于占用内存较多的大对象而言，它的选择可能就不是这样的。因为大对象出现在年轻代很可能扰乱年轻代 GC，并破坏年轻代原有的对象结构。因为尝试在年轻代分配大对象，很可能导致空间不足，为了有足够的空间容纳大对象，JVM 不得不将年轻代中的年轻对象挪到年老代。因为大对象占用空间多，所以可能需要移动大量小的年轻对象进入年老代，这对 GC 相当不利。

基于以上原因，可以将大对象直接分配到年老代，保持年轻代对象结构的完整性，这样可以提高 GC 的效率。如果一个大对象同时又是一个短命的对象，假设这种情况出现很频繁，那对于 GC 来说会是一场灾难。原本应该用于存放永久对象的年老代，被短命的对象塞满，这也意味着对堆空间进行了洗牌，扰乱了分代内存回收的基本思路。因此，在软件开发过程中，应该尽可能避免使用短命的大对象。

可以使用参数
```xml
-XX:PetenureSizeThreshold 
```
设置大对象直接进入年老代的阈值。当对象的大小超过这个值时，将直接在年老代分配。参数-XX:PetenureSizeThreshold 只对串行收集器和年轻代并行收集器有效，并行回收收集器不识别这个参数。

## 设置对象进入年老代的年龄
如何设置对象进入年老代的年龄
堆中的每一个对象都有自己的年龄。一般情况下，年轻对象存放在年轻代，年老对象存放在年老代。为了做到这点，虚拟机为每个对象都维护一个年龄。如果对象在 Eden 区，经过一次 GC 后依然存活，则被移动到 Survivor 区中，对象年龄加 1。以后，如果对象每经过一次 GC 依然存活，则年龄再加 1。当对象年龄达到阈值时，就移入年老代，成为老年对象。

这个阈值的最大值可以通过参数
```xml
-XX:MaxTenuringThreshold 
```
来设置，默认值是 15。虽然-XX:MaxTenuringThreshold 的值可能是 15 或者更大，但这不意味着新对象非要达到这个年龄才能进入年老代。事实上，对象实际进入年老代的年龄是虚拟机在运行时根据内存使用情况动态计算的，这个参数指定的是阈值年龄的最大值。即，实际晋升年老代年龄等于动态计算所得的年龄与-XX:MaxTenuringThreshold 中较小的那个。

## 稳定的 Java 堆 VS 动荡的 Java 堆
一般来说，稳定的堆大小对垃圾回收是有利的。**获得一个稳定的堆大小的方法是使-Xms 和-Xmx 的大小一致，即最大堆和最小堆 (初始堆)一样**

如果这样设置，系统在运行时堆大小理论上是恒定的，稳定的堆空间可以减少 GC 的次数。因此，很多服务端应用都会将最大堆和最小堆设置为相同的数值。但是，一个不稳定的堆并非毫无用处。稳定的堆大小虽然可以减少 GC 次数，但同时也增加了每次GC的时间。让堆大小在一个区间中震荡，在系统不需要使用大内存时，压缩堆空间，使 GC 应对一个较小的堆，可以加快单次 GC 的速度。基于这样的考虑，JVM 还提供了两个参数用于压缩和扩展堆空间。

```xml
-XX:MinHeapFreeRatio
```
参数用来设置堆空间最小空闲比例，默认值是 40。当堆空间的空闲内存小于这个数值时，JVM 便会扩展堆空间。
```xml
-XX:MaxHeapFreeRatio
```
参数用来设置堆空间最大空闲比例，默认值是 70。当堆空间的空闲内存大于这个数值时，便会压缩堆空间，得到一个较小的堆。

当-Xmx 和-Xms 相等时，-XX:MinHeapFreeRatio 和-XX:MaxHeapFreeRatio 两个参数无效。

## 增大吞吐量提升系统性能
吞吐量优先的方案将会尽可能减少系统执行垃圾回收的总时间，故可以考虑关注系统吞吐量的**并行回收收集器**。在拥有高性能的计算机上，进行吞吐量优先优化，可以使用参数：

```xml
java –Xmx3800m –Xms3800m –Xmn2G –Xss128k –XX:+UseParallelGC 
   –XX:ParallelGC-Threads=20 –XX:+UseParallelOldGC
```
–Xmx380m –Xms3800m：设置 Java 堆的最大值和初始值。一般情况下，为了避免堆内存的频繁震荡，导致系统性能下降，我们的做法是设置最大堆等于最小堆。假设这里把最小堆减少为最大堆的一半，即 1900m，那么 JVM 会尽可能在 1900MB 堆空间中运行，如果这样，发生 GC 的可能性就会比较高；

-Xss128k：减少线程栈的大小，这样可以使剩余的系统内存支持更多的线程；

-Xmn2g：设置年轻代区域大小为 2GB；

–XX:+UseParallelGC：年轻代使用并行垃圾回收收集器。这是一个关注吞吐量的收集器，可以尽可能地减少 GC 时间。

–XX:ParallelGC-Threads：设置用于垃圾回收的线程数，通常情况下，可以设置和 CPU 数量相等。但在 CPU 数量比较多的情况下，设置相对较小的数值也是合理的；

–XX:+UseParallelOldGC：设置年老代使用并行回收收集器。

## 尝试使用大的内存分页
CPU 是通过寻址来访问内存的。32 位 CPU 的寻址宽度是 0~0xFFFFFFFF ，计算后得到的大小是 4G，也就是说可支持的物理内存最大是 4G。但在实践过程中，碰到了这样的问题，程序需要使用 4G 内存，而可用物理内存小于 4G，导致程序不得不降低内存占用。为了解决此类问题，现代 CPU 引入了 MMU（Memory Management Unit 内存管理单元）。MMU 的核心思想是利用虚拟地址替代物理地址，即 CPU 寻址时使用虚址，由 MMU 负责将虚址映射为物理地址。MMU 的引入，解决了对物理内存的限制，对程序来说，就像自己在使用 4G 内存一样。内存分页 (Paging) 是在使用 MMU 的基础上，提出的一种内存管理机制。它将虚拟地址和物理地址按固定大小（4K）分割成页 (page) 和页帧 (page frame)，并保证页与页帧的大小相同。这种机制，从数据结构上，保证了访问内存的高效，并使 OS 能支持非连续性的内存分配。在程序内存不够用时，还可以将不常用的物理内存页转移到其他存储设备上，比如磁盘，这就是大家耳熟能详的虚拟内存。

在 Solaris 系统中，JVM 可以支持 Large Page Size 的使用。使用大的内存分页可以增强 CPU 的内存寻址能力，从而提升系统的性能。

```xml
java –Xmx2506m –Xms2506m –Xmn1536m –Xss128k –XX:++UseParallelGC
 –XX:ParallelGCThreads=20 –XX:+UseParallelOldGC –XX:+LargePageSizeInBytes=256m
```
–XX:+LargePageSizeInBytes：设置大页的大小。

过大的内存分页会导致 JVM 在计算 Heap 内部分区（perm, new, old）内存占用比例时，会出现超出正常值的划分，最坏情况下某个区会多占用一个页的大小。

## 使用非占有的垃圾回收器
为降低应用软件的垃圾回收时的停顿，首先考虑的是使用关注系统停顿的 CMS 回收器，其次，为了减少 Full GC 次数，应尽可能将对象预留在年轻代，因为年轻代 Minor GC 的成本远远小于年老代的 Full GC。

```xml
java –Xmx3550m –Xms3550m –Xmn2g –Xss128k –XX:ParallelGCThreads=20
 –XX:+UseConcMarkSweepGC –XX:+UseParNewGC –XX:+SurvivorRatio=8 –XX:TargetSurvivorRatio=90
 –XX:MaxTenuringThreshold=31
```
–XX:ParallelGCThreads=20：设置 20 个线程进行垃圾回收；

–XX:+UseParNewGC：年轻代使用并行回收器；

–XX:+UseConcMarkSweepGC：年老代使用 CMS 收集器降低停顿；

–XX:+SurvivorRatio：设置 Eden 区和 Survivor 区的比例为 8:1。稍大的 Survivor 空间可以提高在年轻代回收生命周期较短的对象的可能性，如果 Survivor 不够大，一些短命的对象可能直接进入年老代，这对系统来说是不利的。

–XX:TargetSurvivorRatio=90：设置 Survivor 区的可使用率。这里设置为 90%，则允许 90%的 Survivor 空间被使用。默认值是 50%。故该设置提高了 Survivor 区的使用率。当存放的对象超过这个百分比，则对象会向年老代压缩。因此，这个选项更有助于将对象留在年轻代。

–XX:MaxTenuringThreshold：设置年轻对象晋升到年老代的年龄。默认值是 15 次，即对象经过 15 次 Minor GC 依然存活，则进入年老代。这里设置为 31，目的是让对象尽可能地保存在年轻代区域。

# 一些监测JVM的命令
- jps 

列出正在运行的虚拟机进程

- jstat  

监视虚拟机运行状态信息

- jmap 

生成堆存储快照

- jstack 

生成虚拟机当前时刻的线程快照，帮助定位线程出现长时间停顿的原因

# 一些JVM参数
## 设定堆内存大小
1. -Xms：启动JVM时的堆内存空间。
2. -Xmx：堆内存最大限制。
3. -Xmn：设置年轻代大小整个堆大小=年轻代大小 + 年老代大小 + 持久代大小，Sun官方推荐配置为整个堆的3/8
4. -XX:PermSize=128M设置持久代大小
5. -XX:MaxPermSize=128M设置持久代最大值，此值可以设置与-XX:PermSize相同，防止持久代内存伸缩，持久代设置很重要，一般预留其使用空间的1/3.

## 设定新生代大小。
1. -XX:NewRatio：新生代和老年代的占比。
2. -XX:NewSize：新生代空间。
3. -XX:SurvivorRatio：伊甸园空间和幸存者空间的占比。
4. -XX:MaxTenuringThreshold：对象进入老年代的年龄阈值。

## 设定垃圾回收器
1. -XX:+UseSerialGC 开启串行收集器
2. -XX:+UseParallelGC开启年轻代并行收集器，JDK5.0以上，JVM会根据系统配置自行设置，所以无需再设置此值
3. -XX:+UseParallelOldGC开启老年代并行收集器
4. -XX:+UseConcMarkSweepGC开启老年代并发收集器(简称CMS)，可以和UseParallelGC一起使用
5. -XX:CMSInitiatingOccupancyFraction=70老年代内存使用比例到多少激活CMS收集器，这个数值的设置有很大技巧基本上满足(Xmx-Xmn)*(100-CMSInitiatingOccupancyFraction)/100>=Xmn否则会出现“Concurrent Mode Failure”，promotionfailed，官方建议数值为68
6. -XX:+UseCMSCompactAtFullCollection：使用并发收集器时，开启对年老代的压缩


## 其他
1. -Xss： 设置每个线程的堆栈大小，设置每个线程的堆栈大小。JDK5.0以后每个线程堆栈大小为1M，以前每个线程堆栈大小为256K。更具应用的线程所需内存大小进行调整。在相同物理内存下，减小这个值能生成更多的线程。但是操作系统对一个进程内的线程数还是有限制的，不能无限生成，经验值在3000~5000左右，这个参数对性能的影响比较大的
2. -XX:MaxTenuringThreshold=0：设置垃圾最大年龄。如果设置为0的话，则年轻代对象不经过Survivor区，直接进入年老代。对于年老代比较多的应用，可以提高效率。如果将此值设置为一个较大值，则年轻代对象会在Survivor区进行多次复制，这样可以增加对象再年轻代的存活时间，增加在年轻代即被回收的概论，linux64的java6默认值是15
3. -XX:ParallelGCThreads=设置并行垃圾回收的线程数。此值可以设置与机器处理器数量相等（逻辑cpu数），这个不确定是物理、还是逻辑使用默认就好
4. -XX:MaxGCPauseMillis=指定垃圾回收时的最长暂停时间，单位毫秒，如果指定了此值的话，堆大小和垃圾回收相关参数会进行调整以达到指定值，设定此值可能会减少应用的吞吐量
5. -XX:GCTimeRatio=设定吞吐量为垃圾回收时间与非垃圾回收时间的比值，公式为1/（1+N）。例如，-XX:GCTimeRatio=19时，表示5%的时间用于垃圾回收。默认情况为99，即1%的时间用于垃圾回收
6. -XX:+UseAdaptiveSizePolicy：设置此选项后，并行收集器会自动选择年轻代区大小和相应的Survivor区比例，以达到目标系统规定的最低相应时间或者收集频率等，此值建议使用并行收集器时，一直打开
7. -XX:+DisableExplicitGC：禁止 java 程序中的 full gc, 如System.gc() 的调用. 最好加上， 防止程序在代码里误用了对性能造成冲击
8. -XX:+PrintGCDetails 打应垃圾收集的情况
9. -XX:+PrintGCTimeStamps  打应垃圾收集的情况
10. -XX:+PrintGCApplicationConcurrentTime：打印每次垃圾回收前，程序未中断的执行时间。可与上面混合使用
11. -XX:+PrintGCApplicationStoppedTime 打应垃圾收集时 , 系统的停顿时间
12. -XX:+PrintGC 打印GC情况
13. -XX:PrintHeapAtGC:打印GC前后的详细堆栈信息

(本文完)

参考：
- http://blog.csdn.net/chen77716/article/details/5695893
- http://www.jianshu.com/p/c6a04c88900a
- https://www.ibm.com/developerworks/cn/java/j-lo-jvm-optimize-experience/index.html
- https://www.dexcoder.com/dexcoder/article/3159
- http://buguoruci.blog.51cto.com/4104173/1287831

