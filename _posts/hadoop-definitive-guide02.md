---
title: 《Hadoop权威指南》书摘-MapReduce概述
link_title: hadoop-definitive-guide02
date: 2018-06-01 14:38:03
tags: [Hadoop]
categories: BigData
thumbnailImage: https://upload-images.jianshu.io/upload_images/79431-923e4afdf70e9574.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/464/format/webp
thumbnailImagePosition: left
---
<span/>
<!-- more -->
![](https://upload-images.jianshu.io/upload_images/79431-923e4afdf70e9574.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/464/format/webp)
<!-- toc -->

**转载请注明出处**
独立博客：http://wangnan.tech 
简书:http://www.jianshu.com/u/244399b1d776**
知乎：https://zhuanlan.zhihu.com/ghoststories

MapReduce是一种可用于数据处理的编程模型，MapReduce程序本质上是并行运行的，因此可以将大规模数据分析任务分发给任何一个拥有足够多机器的数据中心，它的优势在于处理大规模数据

### map和reduce
MapReduce任务可以分为两个处理阶段：map阶段和reduce阶段，每个阶段都以键值对作为输入和输出
![](https://upload-images.jianshu.io/upload_images/79431-afc5d832da3986d4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/719/format/webp)

### 一些概念
**job**
MapReduce job 是客户端需要执行的一个工作单元,它包括输入数据，MapReduce程序和配置信息

**task**
Hadoop将 job 分成若干 task 执行，task包括两类:map任务和reduce任务，这些任务运行在集群的节点上，并通过YARN进行调度，如果任务失败，它将在另一个不同的节点上自动重新调度运行

**split**
hadoop将MapReduce的输入数据划分成等长的小数据块，成为split，hadoop为每个分片创建一个map任务

对于大多数作业来说，一个合理的分片大小趋向于HDFS的一个块的大小，默认是123MB

**数据本地化优化**
Hadoop在储存有输入数据（HDFS中的数据）的节点上运行map任务，可以获得最佳性能，因为它无需使用宝贵的集群带宽资源，但是有时储存该分片的hdfs数据块副本所在的节点可能在运行其他的map任务，此时调度需要从某一个数据块所在的机架中的一个节点上寻找一个空闲的map槽（slot）来运行该map任务分片，仅仅在非常偶然的情况下会发生，这将导致机架之间发生网络传输

**思考题：为什么最近分片大小应该与块大小相同**

**思考题：为什么map任务将其输出写入本地磁盘，而不是HDFS**

### reduce

reduce任务并不具备数据本地化优势，单个reduce任务的输入通常来自于所有mapper的输出，reduce的输出通常储存在HDFS中以实现可靠储存，reduce输出的每个HDFS块，第一个副本储存在本地节点上，其他复本出于可靠性考虑储存在其他机架的节点上

reduce数据流图
![](https://upload-images.jianshu.io/upload_images/79431-145c928440af4cd0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/628/format/webp)

reduce任务的数量并非由输入数据的大小决定，相反是独立的

**partition**
如果有好多个reduce任务，每个map任务就会针对输出进行分区（partition）,即为每个reduce任务建一个分区，每个分区有许多键，但每个建对应的键值对记录都在同一个分区中，分区可以由用户定义的分区函数控制，但通常默认的partitioner通过哈希函数来分区，很高效


![](https://upload-images.jianshu.io/upload_images/79431-4d3b339f5eba475b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/632/format/webp)
**shuffle**
多个reduce任务的数据流如图，map和reduce任务之间的数据流成为shuffle（混洗），因为每个reduce任务的输入都来自许多map任务，shuffle一般比图中所示的更负责，而且调整shuffle参数对作业总执行时间影响比较大

当数据完全可以并行，无需shuffle时，可能出现无reduce任务的情况，这种情况下，唯一的非本地节点数据传输是map任务

**combiner**函数
能帮助减少mapper和reducer之间的数据传输量


