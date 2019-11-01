---
title: 《Hadoop权威指南》书摘-HDFS概述
link_title: hadoop-definitive-guide03
date: 2018-07-26 16:14:32
tags: [Hadoop]
categories: BigData
thumbnailImage: https://i.loli.net/2019/09/24/iaJnuj4XeqwR6C5.png
thumbnailImagePosition: left
---
<span/>
<!-- more -->
![](https://i.loli.net/2019/09/24/iaJnuj4XeqwR6C5.png)
<!-- toc -->

**转载请注明出处**
独立博客：http://wangnan.tech 
简书:http://www.jianshu.com/u/244399b1d776**
知乎：https://zhuanlan.zhihu.com/ghoststories

# 设计
- 超大文件，已经有储存PB级数据的Hadoop集群了
- 流式数据访问，一次写入，多次读取
- 商用硬件 不需要运行在昂贵的硬件上
- 低时间延迟的数据访问
- 大量的小文件
- 多用户写入，任意修改文件

# 概念
## 数据块 chunk
默认为128MB  
HDFS中小于一个快大小的文件不会占据整个快的空间

分块好处：
- 一个文件的大小可以大于网络中任意一个磁盘的容量
- 使用抽象块而非整个文件作为储存单元，大大简化了储存子系统的设计
- 块适合数据备份而提供数据容错性和高可用性

## namenode 和 datanode

管理节点
- 管理文件系统的命名空间，维护这文件系统树内所有文件和目录，还记录这每个文件中各个块所在数据节点的信息
- 没有管理节点，文件系统将无法使用

容错机制：
1. 备份元数据持久状态文件，一般是将持久状态写入本地磁盘的同时，写入一个远程挂载的网络文件系统（NFS）
2. 运行一个辅助namenode,作为主备

工作节点
- 它根据需要储存并检索数据块，并且定期向namenode发送他们所储存的块的列表

## 块缓存
通常datanode从磁盘读取块，但是对于访问频繁的文件，其对应的的块可能被显式地缓存在datanode的内存中，以堆外缓存的形式存在

## 高可用
hadoop配置了一对  active-standy namenode 当活动namenode失效，备用namenode就会接管它的任务并开始服务于来自客户端的请求，不会有任何明显的中断

# 数据流
## 文件读取
![](https://upload-images.jianshu.io/upload_images/79431-eddcdd26303a9789.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/628/format/webp)

- 客户端通过RPC调用namenode，以确定文件起始块的位置,对于每一个块，namenode返回存在该快副本的datanode地址，此外，这些datanode根据他们与客户端的距离来排序，如果该客户端本身就是一个datanode，那么该客户端将会从保存有相应数据块复本的本地datanode读取数据


## 文件写入
![](https://upload-images.jianshu.io/upload_images/79431-fc8fa64892ce3f33.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/641/format/webp)

- 客户端通过RPC调用namenode，在文件系统的命名空间中新建一个文件，此时该文件中还没有相应的数据块
- namenode执行各种不同的检查以确保这个文件不存在以及客户端有新建该文件的权限，如果通过，就为创建新文件记录一条记录
- 写入时，将数据分成一个个数据包，并写入内部队列，挑选出适合存储数据复本的一组datanode，并据此来要求namenode分配新的数据块

思考题：namenode选择再哪个datanode储存复本
