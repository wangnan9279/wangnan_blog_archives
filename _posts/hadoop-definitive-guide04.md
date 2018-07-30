---
title: 《Hadoop权威指南》书摘-关于YARN
link_title: hadoop-definitive-guide04
date: 2018-07-27 14:55:32
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
知乎：https://zhuanlan.zhihu.com/c_121958856

# 简介
Apache YARN (Yet Another Resource Negotiaor的缩写)是Hadoop的集群资源管理系统，YARN被引入Hadoop2，最初是为了改善MapReduce的实现，但他具有足够的通用性，同样可以支持其他分布式计算模型

YARN提供请求和使用集群资源的API，但这些API很少直接用于用户代码，用户代码用的是分布式计算框架提供的更高层API,如图，一些分布式计算框架（MapReduce Spark）作为YARN应用运行在集群计算层（YARN）和集群储存层（HDFS和Hbase）上

![](http://onxkn9cbz.bkt.clouddn.com/hadoop04/01.png)

# 运行机制
YARN通过两类长期运行的守护进程提供自己的核心服务：
- 资源管理器（resource manager） 管理集群上资源使用
- 节点管理器（noede manager）运行在集群中所以节点上且能够启动和监控容器

**容器**
容器用于执行特定应用程序的进程，每个容器都有资源限制（内存，cpu等）一个容器可以是一个unix进程，也可以是一个linux cgroup 取决于yarn的配置

![](http://onxkn9cbz.bkt.clouddn.com/hadoop04/02.png)

为了在yarn上运行一个应用，首先，客户端联系资源管理器，要求它运行一个application master进程，然后，资源管理器找到一个能够子啊容器中启动application master 的节点管理器

application master一旦运行起来做些什么依赖于应用本身，所以可以看出，yarn本身不会为应用的各部分（client,master和进程）彼此通信提供任何手段

## 资源请求
yarn有一个灵活的资源请求模型，当请求多个容器时，可以指定每个容器需要的计算机资源数量（内存和cpu），还可以指定对容器本地限制要求

yarn应用可以在运行中的任意时刻提出资源申请，例如可以
1. 在最开始提出所有的请求
2. 或者为了满足不断变化的应用需要，采取更为动态的方式在需要更多资源时提出请求

spark采用了第一种方式，MapReduce则分两步走，在最开始的时候申请map任务容器，reduce任务容器的启动则放在后期，同样，如果任何任务出现失败，将会另外申请容器重新运行失败的任务

## 应用生命期
最简单的模型是一个用户作业对应一个应用，这也是MapReduce采取的方式

第二种模型是，作业的每个工作流或者每个用户对话对应一个应用，这种模型比第一种效率高，因为容器可以在作业之间重用，并且有可能缓存作业之间的中间数据

第三种模型，多个用户共享一个长期运行的应用，这种应用通常是作为一种协调者的角色运行的，或者是一种代理应用，由于避免了启动新的application master带来的开销，一个总是开启的application master意味着用户将获得非常低延迟的查询响应

## 构建YARN应用
从无到有编写一个yarn应用是一件非常复杂的事情，在很大情况下不必这样，有很多现成的应用，在符合要求的情况下可以直接使用，例如使用spark tez storm等

# YARN和MapReduce对比
- 可扩展性
- 可用性
- 利用率
- 多租户

# YARN中的调度
yarn调度器的工作就是根据既定策略为应用分配资源

## 调度选项
YARN有三种调度器
- FIFO调度器
- 容量调度器
- 公平调度器

![](http://onxkn9cbz.bkt.clouddn.com/hadoop04/03.png)


**FIFO调度器**
将应用防止正在一个队列中，然后按照提交的顺序运行应用

优点：简单易懂，不需要配置
缺点: 不适合共享集群，大的应用会占用集群的所有资源呢，

在一个共享集群中更时候使用容量或者公平调度器，他们允许长时间允许的作业可以及时完成，同事允许正在进行的较小的临时查询的用户能够在合理的时间内得到返回结果

**容量调度器**
一个独立的专门的队列保证小作业可以一提交就可以启动，
缺点：这个策略是以整个集群的利用率为代价的，意味着大作业执行的时间要更长

**公平调度器**
优点：不需要预留一定量的资源，因为调度器会在所有运行的作业之间动态平衡资源
得到了较高的集群利用率，又能保证小作业能及时完成

## 调度器配置
### 容量调度器配置

通过以下属性：
yarn.scheduler.capacity.<queue-path>.<sub-property>


队列的配置：
在MapReduce中可以通过：mapreduce.job.queuename来指定要用的队列

### 公平调度器配置

由属性：yarn.resourcemanager.scheduler.class的设置决定，默认是使用容量调度器，如果要使用公平调度器，需要将yarn-site.xml文件中的yarn.resourcemanager.scheduler.class设置为公平调度器的完全限定名：org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.FairScheduler

## 延迟调度
如果等待一小段时间（不超过几秒）能够戏剧性的增加在所有请求的节点上分配到一个容器的机会，从而提高集群的效率，这个特性称为延迟调度

容量调度器和公平调度器都支持延迟调度

原因：
yarn中的每个节点管理器周期性（默认每秒一次）向资源管理器发送心跳请求，心跳中携带了节点管理器中正在运行的容器，新容器可用的资源等信息，这样对于一个计划运行一个容器的应用而言，每个心跳就是一个潜在的调度机会

## 主要资源公平性
yarn中调度会观察每个用户的主导资源（cpu or 内存），并将其作为怼集群资源使用的一个度量

