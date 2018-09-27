---
title: 《Hadoop权威指南》书摘-初识Hadoop
link_title: hadoop-definitive-guide01
date: 2018-02-12 13:43:28
tags: [Hadoop]
categories: BigData
thumbnailImage: http://onxkn9cbz.bkt.clouddn.com/hadoop/hadoop.png
thumbnailImagePosition: left
---
<span/>
<!-- more -->
![](http://onxkn9cbz.bkt.clouddn.com/hadoop/hadoop.png)
<!-- toc -->

**转载请注明出处**
独立博客：http://wangnan.tech 
简书:http://www.jianshu.com/u/244399b1d776**
知乎：https://zhuanlan.zhihu.com/ghoststories

## 数据！数据！
我们生活在一个数据爆炸的时代，我们必须想办法好好的的储存和分析这些数据

## 数据储存和分析
1. 解决硬件故障：使用副本
2. 解决从无数个硬盘中读取数据到一起去分析：MapReduce编程模型

hadoop是什么？简而言之，hadoop为我们提供了一个可靠且可扩展的储存和分析平台

## 查询所有数据
MapReduce是一个批量查询处理器，能够在合理的时间范围内处理针对整个数据集的动态查询

## 不仅仅是批处理
MapReduce更适合那种没有用户在现场等待查询结果的离线使用场景

Hadoop的发展已经超越了批处理本身，实际上，名词"Hadoop"有时被用于指代一个更大的，多项目组成的生态系统，产生了一些可以与hadoop协同工作的处理模式，比如交互式SQL、迭代处理、流处理、搜索，项目例子：Hbase、YARN、Hive、Spark、Storm、Solr

## 相较其他系统的优势
1. 关系型数据库
![](http://onxkn9cbz.bkt.clouddn.com/hadoop01/01.png)


2. 网格计算

- Hadoop尽量在计算节点上储存数据，以实现数据的本地快速访问

- MapReduce任务之间是彼此独立的，框架能够检测到失败的任务并重新再正常的机器上执行，任务的执行顺序也无关紧要

3. 志愿计算
MapReduce有三大设计目标：
- 为只需短短几分钟或几小时就可以完成的作业提供服务
- 运行于同一个内部有高速网络连接的数据中心内
- 数据中心内的计算机都是可靠的、专门的硬件

## Hadoop发展简史
- Hadoop是lucene创始人Doug Cutting创建的
- 起源于开源网络搜索引擎Apache Nutch
- 关于Hadoop名字的来历，Doug这样解释：这个名字是我的孩子给他的毛绒象玩家取的，我的命名标准就是好拼读，含义宽泛，不会用于其他地方，小朋友是这方面的高手，Googo！就是他们起的
- 2008年成为Apache顶级项目
- 目前Hadoop被主流企业广泛使用，在工业界，Hadoop已经是公认的大数据通用和分析平台
