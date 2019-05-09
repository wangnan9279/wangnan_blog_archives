---
title: Elasticsearch底层索引控制
tags: [ElasticSearch]
date: 2016-10-03 17:44:54
categories: ElasticSearch
link_title: elasticsearch-index-control
thumbnailImage: https://upload-images.jianshu.io/upload_images/79431-c6d398af52f4b48c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/558/format/webp
thumbnailImagePosition: left
---
<!-- toc -->
<!-- more -->
![](https://upload-images.jianshu.io/upload_images/79431-c6d398af52f4b48c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/558/format/webp)
>   如何使用不同评分公式及其特性
    如何使用不同的倒排表格式极其特性
    如何处理准实时搜索、实时读取、以及搜索器重新打开之后的动作
    配置搜索事务日志以满足应用需求，并查看它对部署的影响
       
#  如何使用不同评分公式及其特性 
2012年Apache lucene 4.0发布之后，用户便可以改变默认的基于TF/IDF的评分算法了，这是因为lucene的API做了一些改变，使得用户能轻松地修改和扩展该评分公式

新增的相似度模型：
- Okipi BM25模型
    - 一种基于概率模型的相似度模型，可用于估算文档与给定查询匹配的概率
    - **在短文本文档上的效果最好**，因为这种场景中重复词项对文档的总体得分损害较大
- 随机偏离模型(Divergence from randomness)
    - 是一种基于同名概率模型相似度的模型
    - 随机偏离模型在**类似自然语言的文本上效果较好**
- 基于信息的模型(Infomation based)
    - 与随机偏离模型类型，也在**类似自然语言的文本上拥有较好的效果**
    
## 为每个字段配置相似度模型
我们希望name字段使用BM25模型，我们需要添加**similarity**字段
![](elasticsearch-index-control/01.png)

# 如何使用不同的倒排表格式极其特性
lucene 4.0的另外一个显著改变就是允许用户改变索引文件编码方式，在此之前，只能通过修改lucene内核代码来实现，而lucene 4.0出现后，它提供了灵活的索引方式，允许用户改变倒排索引的写入方式

为什么用户需要修改lucene索引写入格式？理由之一是性能

# 为每个字段配置编解码器
需要在字段配置文件中添加一个**postings_format**属性，并将具体的编解码器所对应的属性值赋给它
![](elasticsearch-index-control/02.png)

# 有哪些倒排表格式
- default 
    - 默认的格式，该格式提供了储存字段和词向量压缩功能
- puling 
    -  该编码器将包含大量不同值的字段的倒排表编码为词项数组
    - 这会减少lucene在搜索文档时的查找操作，可以提高这种字段的搜索速度
- direct 
    - 在读索引阶段将词项载入词典，且词项在内存中为未压缩状态，
    - 能提升常用字段的查询性能，但也需要谨慎使用，非常消耗内存
- memory
    - 改解编码器将所有数据写入磁盘
- bloom_default     
    - 是default编码器的一种扩展
    - 用户快速判断某个值是否存在

#  如何处理准实时搜索、实时读取、以及搜索器重新打开之后的动作
在索引期新文档会写入索引段，索引段是独立的lucene索引，这意味着查询时可以与索引并行的，只是不会有新增的索引段被添加到可被搜索的索引段集合之中

Apache lucene通过创建后续的segment_N文件来实现此功能，且该文件列举了索引中的索引段，这个过程称之为提交(committing)

Lucene以一种安全的方式来执行该操作，能确保索引更改以原子操作方式写入索引

尽管我们向索引中添加了文档，但它并没有执行提交commit操作，

lucene使用了一个叫做searcher的抽象类类执行新索引段的加入

searcher重新打开的过程叫做刷新（refresh）

在例子中可以使用下面的命令：

    curl -xGET localhost:9200/test/_refresh

## 更改默认的刷新时间
可以更改elasticsearch中的index.refresh_interval参数或者使用配置更新相关的api

刷新操作是耗资源的，因此刷新间隔时间越长，索引速度越快

如果需要长时间高速建索引，并且在建索引结束之前暂时不执行查询，那么可以考虑将index.refresh_interval参数设置为-1.然后在建索引结束以后再将改参数恢复为默认值

## 事务日志
lucene 能保证索引的一致性，但是这并不能保证当往索引中写数据失败时不会损失数据，；另外，频繁提交操作会导致严重的性能问题

elasticsearch通过使用 事务日志（transaction log）来解决这些问题，它能保存所有未提交的事务，而es会不时创建一个新的日志文件用于记录每个事务的后续操作，当有错误发生时就会检查事务日志，必要是会再次执行某些操作，以确保没有丢失任何更改的信息

事务日志中的信息与存储介质之间的同步，称为**事务日志刷新（flushing）**

除了自动的事务日志刷新以外，也可以使用对应的api

    curl -XGET localhost:9200/flush


### 事务日志相关配置
- **index.translog.flush_threshold_period**  该参数的默认值为30分钟，它控制了强制自动事务日志刷新的时间间隔
- **index.translog.flush_threshold_ops**
该参数确定了一个最大操作数，在上一次事务日志刷新以后，当索引更改操作次数超过了该参数值时，强制进行事务日志刷新操作，默认为5000
- **index.trans.flush_threshold_size**
该参数确定了事务日志的最大容量，当容量大于等于该参数值，就强行进行事务日志刷新操作，默认为200MB
- **index.translog.disable_flush**
禁用事务日志刷新

（注：内容整理自《深入理解elasticsearch》）