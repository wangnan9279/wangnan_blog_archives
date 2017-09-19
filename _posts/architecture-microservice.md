---
title: 微服务架构解析(附思维导图)
link_title: architecture-microservice
date: 2017-08-30 16:31:29
tags: []
categories: 架构
thumbnailImage: http://onxkn9cbz.bkt.clouddn.com/41.jpg	
thumbnailImagePosition: left
---
<span/>
<!-- more -->
![](http://onxkn9cbz.bkt.clouddn.com/41.jpg)
<!-- toc -->
# 思维导图
![](http://onxkn9cbz.bkt.clouddn.com/microservices01.png)
![](http://onxkn9cbz.bkt.clouddn.com/microservices03.png)


# 介绍
微服务架构（Microservice Architecture）是一种架构概念

旨在通过将功能分解到各个离散的服务中以实现对解决方案的解耦

将功能分解到离散的各个服务当中，从而降低系统的耦合性，并提供更加灵活的服务支持。

# 传统开发模式和微服务的区别
## 优点
- 开发简单，集中式管理
- 基本不会重复开发
- 功能都在本地，没有分布式的管理和调用消耗

## 缺点
1. 效率低：开发都在同一个项目改代码，相互等待，冲突不断
2. 维护难：代码功功能耦合在一起，新人不知道何从下手
3. 不灵活：构建时间长，任何小修改都要重构整个项目，耗时
4. 稳定性差：一个微小的问题，都可能导致整个应用挂掉
5. 扩展性不够：无法满足高并发下的业务需求

# 微服务架构特征
## 官方的定义：
1. 一些列的独立的服务共同组成系统
2. 单独部署，跑在自己的进程中
3. 每个服务为独立的业务开发
4. 分布式管理
5. 非常强调隔离性

## 大概的标准
1. 分布式服务组成的系统
2. 按照业务，而不是技术来划分组织
3. 做有生命的产品而不是项目
4. 强服务个体和弱通信（ Smart endpoints and dumb pipes ）
5. 自动化运维（ DevOps ）
6. 高度容错性
7. 快速演化和迭代

# SOA和微服务的区别
- SOA喜欢重用，微服务喜欢重写
- SOA喜欢水平服务，微服务喜欢垂直服务
- SOA喜欢自上而下，微服务喜欢自下而上

# 实践微服务
## 客户端如何访问这些服务
一般在后台N个服务和UI之间一般会一个代理或者叫API Gateway

作用：
- 提供统一服务入口，让微服务对前台透明
- 聚合后台的服务，节省流量，提升性能
- 提供安全，过滤，流控等API管理功能

## 每个服务之间如何通信
- REST（JAX-RS，Spring Boot）
- RPC（Thrift, Dubbo）

## 服务发现服务注册
- zookeeper
- dubbo

## 服务挂了，如何解决
- 重试机制
- 限流
- 熔断机制
- 负载均衡
- 降级（本地缓存）

参考：Netflix的Hystrix

# 优缺点
- 优点
复杂度可控，独立按需扩展，技术选型灵活，容错，可用性高
- 缺点
多服务运维难度，系统部署依赖，服务间通信成本，数据一致性，系统集成测试，重复工作，性能监控等

# 思考
微服务对我们的思考，更多的是思维上的转变。对于微服务架构：技术上不是问题，意识比工具重要。

关于微服务的几点设计出发点：
1. 应用程序的核心是业务逻辑，按照业务或客户需求组织资源（这是最难的）
2. 做有生命的产品，而不是项目
3. 头狼战队，全栈化
4. 后台服务贯彻Single Responsibility Principle（单一职责原则）
5. VM->Docker （to PE）
6. DevOps (to PE)

同时，对于开发同学，有这么多的中间件和强大的PE支持固然是好事，我们也需要深入去了解这些中间件背后的原理，知其然知其所以然，在有限的技术资源如何通过开源技术实施微服务？

最后，一般提到微服务都离不开DevOps和Docker，理解微服务架构是核心，devops和docker是工具，是手段。


# 参考
http://www.cnblogs.com/imyalost/p/6792724.html?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io

http://www.jianshu.com/p/77ce2dbd1d6e

http://kb.cnblogs.com/page/520922/

http://www.infoq.com/cn/articles/seven-uservices-antipatterns

http://www.csdn.net/article/2015-08-07/2825412

http://blog.csdn.net/mindfloating/article/details/45740573

http://blog.csdn.net/sunhuiliang85/article/details/52976210

http://www.oschina.net/news/70121/microservice
