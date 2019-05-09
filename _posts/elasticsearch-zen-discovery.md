---
title: Elasticsearch5.5官方文档翻译-Zen Discovery
link_title: elasticsearch-zen-discovery
date: 2017-08-21 09:40:54
tags: [ElasticSearch]
categories: ElasticSearch
thumbnailImage: https://upload-images.jianshu.io/upload_images/79431-875d1ea42fb5b528.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/510/format/webp	
thumbnailImagePosition: left
---
<span/>
<!-- more -->
![](https://upload-images.jianshu.io/upload_images/79431-875d1ea42fb5b528.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/510/format/webp)
<!-- toc -->
# 章节
Modules » Discovery » Zen Discovery
# 概述


Zen Discovery是内置在elasticsearch的默认发现模块。它提供单播发现，但可扩展到支持云环境和其他形式的发现。

禅发现集成了其它模块，例如，节点之间的所有通信是使用transport模块。

它被分离成多个子模块，其解释如下：

- Ping
这是一个节点使用发现机制找到其他节点的过程。

- Unicast 
单播发现需要一个主机列表，用于将作为GossipRouter。这些宿主可被指定为主机名或IP地址;指定主机名的主机每一轮Ping过程中解析为IP地址。

需要注意的是，因为有Java安全管理器，JVM的默认无限期缓存有效的主机名解析。这可以通过添加修改 
```
networkaddress.cache.ttl=<timeout>
```
到你的 Java安全策略。未能解析的任何主机都将被记录。

还要注意的是，因为有Java安全管理器，JVM的默认缓存无效的主机名解析十秒钟。这可以通过添加修改 
```
networkaddress.cache.negative.ttl=<timeout> 
```
到你的Java安全策略。

建议在单播的主机列表中维护集群中的主合格节点的列表。

单播的发现提供了与以下设置discovery.zen.ping.unicast前缀：

- hosts
用数组阵列设置或逗号分隔设置。每个值应在的形式，host:port或host（其中port默认为设置transport.profiles.default.port 如果未设定使用transport.tcp.port）。需要注意的是IPv6的主机必须用括号括起来。默认为127.0.0.1, [::1]

- hosts.resolve_timeout

每轮ping的DNS查找等待时间量。默认为5秒。

单播发现使用传输模块来执行发现。

# Master选举
作为Ping过程的一部分，集群的主节点要么当选要么加入假期。这是自动完成的。
```
discovery.zen.ping_timeout（默认为3s）
```
允许选举时间的波动去处理速度慢或拥塞网络的情况（较高的值保证失败率少）。一旦一个节点加入时，它会发出一个加入
请求到主（discovery.zen.join_timeout）超时时间默认是ping timeout的20倍

当主节点停止或遇到问题，群集节点萌重新开始执行ping命令，并会选出新的主节点。这轮Ping也可对抗（部分）网络故障，其中一个节点可能认为主节点挂掉。在这种情况下，节点将从其他节点获取当前的主节点

如果
```
discovery.zen.master_election.ignore_non_master_pings
```
是true，选主期间，从没有资格的节点（当节点的node.master是false）奖杯忽略
该值默认值是 false。


节点可以被排除成为主节点当node.master为false。

```
discovery.zen.minimum_master_nodes
```
设置需要加入一个新的主选举的最小的主合格节点数量，相同的设置控制活的主合格节点的最小数量，应该是活集群的一部分。如果这一要求得不到满足，活动主节点将下台，新的主导选举将开始。

此设置必须通过一个公式算出来。建议避免仅具有两个主资格的节点中，因为任何一个主合格的节点的丢失将导致不可操作群集。

# 故障检测
有两个故障检测运行过程。首先是由主，来ping集群中的所有其他节点，并验证它们是否还活着。而在另一端，每个节点ping 主节点，以验证它是否还活着或选举过程需要启动。

以下设置控制使用的故障检测过程 以
```
discovery.zen.fd
```
作前缀：

- ping_interval
多久一个节点被ping。默认为1s。

- ping_timeout
等待多久ping响应，默认为 30s。

- ping_retries
有多少ping失败/超时导致被视为一个节点失败。默认为3。

# 集群状态
主节点是群集中，可以使改变集群状态的唯一节点。主节点一次处理一个集群状态更新，状态改变和发布更新的到集群中的所有其他节点。每个节点接收发布消息，确认它，但还没有立即应用它。如果主节点没有接收来自节点确认的数量至少为discovery.zen.minimum_master_nodes，在时间（由受控discovery.zen.commit_timeout设置，默认值为30秒）内。节点集群状态改变被拒绝。

一旦足够的节点已作出回应，集群状态改变被提交然后消息将被发送到所有结点。然后节点然后进行新的群集状态适用于他们的内部状态。主节点等待所有节点响应，在去队列处理下一个状态更新之前，直到超时，超时时间是在discovery.zen.publish_timeout默认情况下设置为30秒，时间从发布开始时测量h超时设置可以通过动态的改变集群更新设置API

# 无主块
为了让集群全面运作，必须有一个有效的主节点和一些主资格的节点。主资格的节点必须满足的数目 discovery.zen.minimum_master_nodes，如果被设置。
```
discovery.zen.no_master_block
```
设置控制什么操作被拒绝，当没有有效的主节点时。

该discovery.zen.no_master_block设置有两个有效的选项：

- all

对节点的所有的操作，比如读与写操作，将被拒绝。这也适用于读与写集群状态的api，获取索引设置，设置mapping和集群状态api

- write（默认）
写操作将被拒绝。读操作会成功。基于最后已知的群集配置。这可能导致局部陈旧数据的读取，因为节点可能从集群的其余部分分离。

该discovery.zen.no_master_block设置并不适用于基于节点的API（例如群集的统计信息，节点信息和节点统计的API）。请这些API将不会被阻塞，可以在任何可用的节点上运行。