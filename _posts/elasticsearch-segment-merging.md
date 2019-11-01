---
title: Elasticsearch段合并
tags: [ElasticSearch]
date: 2016-10-08 15:41:03
categories: ElasticSearch
link_title: elasticsearch-segment-merging
thumbnailImage: https://i.loli.net/2019/09/25/wOcjeg83JZbDn2E.png
thumbnailImagePosition: left
---
<!-- toc -->
<!-- more -->
![](https://i.loli.net/2019/09/25/wOcjeg83JZbDn2E.png)
> elasticsearch 中每个索引都会创建一个到多个**分片**和零个到多个**副本**，这些分片或副本实质上都是**lucene索引**


**lucene索引**是基于多个索引段创建，索引文件中绝大部分数据都是只写一次，读多次，而只有用于保存文档删除信息的文件才会被多次更改

在某些时刻，当某种条件满足时，多个索引段会被拷贝合并到一个更大的索引段，而那些旧的索引段会被抛弃并从磁盘中删除，这操作叫做段合并（segment merging）


# 为什么要进行段合并？
- 索引段的个数越多，搜索性能越低且消耗内存更多
- 索引段是不可变的，物理上你并不能从中删除信息（如果你碰巧从索引中删除了大量文档，但这些文档只是做了删除标记，物理上并没有被删除）而当段合并发送时，这些标记为删除的文档并没有被复制到新的索引段中 

# 段合并好处
- 当多个索引段合并为一个的时候，会减少索引段的数量并提高搜索速度
- 同时也会减少索引的容量（文档数）

# 段合并代价
- IO操作代价，在速度较慢的系统中，段合并会显著影响性能

elasticsearch允许用户选择段合并政策（merge policy）及储存级节流（store level throttling）

# 选择正确的段合并策略
尽管段合并是lucene的责任，elasticsearch也允许用户配置想用的段合并策略
到目前为止有三种可用的合并策略：
- **tiered（默认）**
它能合并大小相似的索引段，并考虑每层允许的索引段的最大个数
- **log_byte_size**
该策略不断地以字节数的对数为计算单位，选择多个索引来合并创建新索引
- **log_doc**
与log_byte_size类似，不同的是前者基于索引的字节数计算，后者基于索引段文档数计算

为了告知elasticsearch我们想使用的段合并策略，可以将配置文件的index.merge.policy字段泪痣成我们期望的段合并策略类型例如：**index.merge.policy.type: tiered**
    

# 调度
es允许我们定制合并策略的执行方式，调度器分两种
默认的是并发合并调度器  ConcurrentMerge-Scheduler
## 并发合并调度器
该调度器使用多线程执行索引合并操作
## 顺序合并调度器
它使用同一个线程执行所有的索引合并操作，在执行合并时，该线程的其他文档处理都会被挂起，从而索引操作会延迟进行

（注：内容整理自《深入理解elasticsearch》）

