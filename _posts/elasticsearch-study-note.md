---
title: Elasticsearch学习笔记
tags: [ElasticSearch]
date: 2016-08-13 14:51:47
categories: ElasticSearch
link_title: elasticsearch-study-note
thumbnailImage: http://onxkn9cbz.bkt.clouddn.com/elasticsearch.png
thumbnailImagePosition: left
---
<!-- toc -->
<!-- more -->
![](http://onxkn9cbz.bkt.clouddn.com/elasticsearch.png)
> Elasticsearch是一个可伸缩的开源全文搜索和分析引擎，它使你可以快速且接近实时的去保存，查询和分析海量的数据，他的潜在应用场景是作为一些有复杂搜索功能和需求的应用的搜索
引擎

# 简介
Elasticsearch是一个基于Apache Lucene(TM)的开源搜索引擎。无论在开源还是专有领域，Lucene可以被认为是迄今为止最先进、性能最好的、功能最全的搜索引擎库。
但是，Lucene只是一个库。想要使用它，你必须使用Java来作为开发语言并将其直接集成到你的应用中，更糟糕的是，Lucene非常复杂，你需要深入了解检索的相关知识来理解它是如何工作的。Elasticsearch也使用Java开发并使用Lucene作为其核心来实现所有索引和搜索的功能，但是它的目的是通过简单的RESTful API来隐藏Lucene的复杂性，从而让全文搜索变得简单。



## 与关系型数据库对比
Relational DB -> Databases -> Tables -> Rows -> Columns
Elasticsearch  -> Indices     -> Types   -> Documents -> Fields

Elasticsearch集群可以包含多个索引(indices)（数据库），每一个索引可以包含多个类型(types)（表），每一个类型包含多个文档(documents)（行），然后每个文档包含多个字段(Fields)（列）

# 基础概念
near realtime（NRT）
es是一个接近实时的搜索平台，这意味着你查询一个文档的时候有一个延时。大约一秒

## cluster
集群是一个或多个节点（服务器）的集合在一起，保存所有的数据，联合所有节点一起提供查询能力。
一个集群有一个唯一的名字，默认是“elasticsearch",集群名很重要，因为集群节点加入集群的唯一方式是根据这个名字。

## node 
节点的默认名字是漫威的一个角色，默认加入集群elasticsearch

## index
索引是一系列具有相似特点文档的集合
实际上，索引只是一个用来指向一个或多个分片(shards)的“逻辑命名空间(logical namespace)

「索引」含义的区分
你可能已经注意到索引(index)这个词在Elasticsearch中有着不同的含义，所以有必要在此做一下区分:
索引（名词） ：如上文所述，一个索引(index)就像是传统关系数据库中的数据库，它是相关文档存储的地方，index的复数是indices 或indexes。
索引（动词） ：「索引一个文档」表示把一个文档存储到索引（名词）里，以便它可以被检索或者查询。这很像SQL中的INSERT关键字，差别是，如果文档已经存在，新的文档将覆盖旧的文档。

倒排索引 传统数据库为特定列增加一个索引，例如B-Tree索引来加速检索。Elasticsearch和Lucene使用一种叫做倒排索引(inverted index)的数据结构来达到相同目的。

## Type
索引中，类型是一种逻辑的分类，它的意义由使用者来赋予

## mapping
>每个类型(type)都有自己的映射(mapping)或者结构定义，就像传统数据库表中的列一样。所有类型下的文档被存储在同一个索引下，但是类型的映射(mapping)会告诉Elasticsearch不同的文档如何被索引
Elasticsearch支持以下简单字段类型：
类型	表示的数据类型
String	string
Whole number	byte, short, integer, long
Floating point	float, double
Boolean	boolean
Date	date

## document
文档是搜索信息的基本单元，用json表达，文档必须被包含于一个type中

>   文档 ID文档唯一标识由四个元数据字段组成：
    _id：文档的字符串 ID
    _type：文档的类型名
    _index：文档所在的索引
    _uid：_type 和 _id 连接成的 type#id

默认情况下，_uid 是被保存（可取回）和索引（可搜索）的。_type 字段被索引但是没有保存，_id 和_index 字段则既没有索引也没有储存，它们并不是真实存在的。

## shards&replicas
es提供能力，让你把index分成好几个部分，叫做分片，当你创建索引的时候，你可以简单的定义分片的个数，每个分片本身是一个独立的功能齐全的“索引”，可以被放到任何的集群节点中

分片的意义：
1.可以水平分割和扩展数据 
2.可以把操作分配给多个分区，提高性能

es允许你制作一个或多个分片的副本。叫做复制分片
复制分片的意义：
1、他提供了高可用性，副本和原始分区不处于一个节点中。
2.他提高了性能，因为搜索可以在任何分区上允许。
每一个分片是一个lucene索引，每个Lucene实例有一个最大的存放文档的数量。这个数量是2417483519

## analysis 
分析也称分词
Elasticsearch中的数据可以大致分为两种类型：
确切值 及 全文文本。
确切值是确定的，正如它的名字一样。比如一个date或用户ID，也可以包含更多的字符串比如username或email地址。
全文文本常常被称为非结构化数据，而对于全文数据的查询来说，却有些微妙。我们不会去询问这篇文档是否匹配查询要求？。 但是，我们会询问这篇文档和查询的匹配程度如何？。换句话说，对于查询条件，这篇文档的相关性有多高？

为了方便在全文文本字段中进行这些类型的查询，Elasticsearch首先对文本分析(analyzes)，然后使用结果建立一个倒排索引。

分析(analysis)机制用于进行全文文本(Full Text)的分词，以建立供搜索用的反向索引。

分析(analysis)是这样一个过程：
首先，标记化一个文本块为适用于倒排索引单独的词(term)
然后标准化这些词为标准形式，提高它们的“可搜索性”或“查全率”

这个工作是分析器(analyzer)完成的。一个分析器(analyzer)只是一个包装用于将三个功能放到一个包里：
字符过滤器
1.首先字符串经过字符过滤器(character filter)，它们的工作是在标记化前处理字符串。字符过滤器能够去除HTML标记，或者转换"&"为"and"。
分词器
2.下一步，分词器(tokenizer)被标记化成独立的词。一个简单的分词器(tokenizer)可以根据空格或逗号将单词分开（译者注：这个在中文中不适用）
标记过滤
3.最后，每个词都通过所有标记过滤(token filters)，它可以修改词（例如将"Quick"转为小写），去掉词（例如停用词像"a"、"and"、"the"等等），或者增加词（例如同义词像"jump"和"leap"）


index参数控制字符串以何种方式被索引。它包含以下三个值当中的一个：
analyzed首先分析这个字符串，然后索引。换言之，以全文形式索引此字段。not_analyzed索引这个字段，使之可以被搜索，但是索引内容和指定值一样。不分析此字段。no不索引这个字段。这个字段不能为搜索到。
string类型字段默认值是analyzed。如果我们想映射字段为确切值，我们需要设置它为not_analyzed：
{
    "tag": {
        "type":     "string",
        "index":    "not_analyzed"
    }
}其他简单类型（long、double、date等等）也接受index参数，但相应的值只能是no和not_analyzed，它们的值不能被分析。

Elasticsearch提供很多开箱即用的字符过滤器，分词器和标记过滤器。这些可以组合来创建自定义的分析器以应对不同的需求。

string类型的字段，默认的，考虑到包含全文本，它们的值在索引前要经过分析器分析，并且在全文搜索此字段前要把查询语句做分析处理。
对于string字段，两个最重要的映射参数是index和analyer。


## highlight
很多应用喜欢从每个搜索结果中高亮(highlight)匹配到的关键字，这样用户可以知道为什么这些文档和查询相匹配。

## score
每个节点都有一个_score字段，这是相关性得分(relevance score)，它衡量了文档与查询的匹配程度。默认的，返回的结果中关联性最大的文档排在首位；这意味着，它是按照_score降序排列的。这种情况下，我们没有指定任何查询，所以所有文档的相关性是一样的，因此所有结果的_score都是取得一个中间值1
max_score指的是所有文档匹配查询中_score的最大值。

## refresh
默认情况下，每个分片每秒自动刷新一次。这就是为什么说Elasticsearch是近实时的搜索了：文档的改动不会立即被搜索，但是会在一秒内可见。

## sort
排序
默认情况下，结果集会按照相关性进行排序 -- 相关性越高，排名越靠前
字段值排序:
按时间排序

    GET /_search
    {
     "query" : {
         "filtered" : {
            "filter" : { "term" : { "user_id" : 1 }}
        }
    },
    "sort": { "date": { "order": "desc" }}
    }

返回：

    "hits" : {
        "total" :           6,
    "max_score" :       null, <1>
    "hits" : [ {
        "_index" :      "us",
        "_type" :       "tweet",
        "_id" :         "14",
        "_score" :      null, <1>
        "_source" :     {
             "date":    "2014-09-24",
             ...
        },
        "sort" :        [ 1411516800000 ] <2>
    },
    ...
    }
    
_score 是比较消耗性能的, 而且通常主要用作排序 -- 我们不是用相关性进行排序的时候，就不需要统计其相关性
字段值默认以顺序排列（从小到大 ），而 _score 默认以倒序排列。

## 缓存
过滤器是怎么计算的。它们的核心是一个字节集来表示哪些文档符合这个过滤器。Elasticsearch 主动缓存了这些字节集留作以后使用。一旦缓存后，当遇到相同的过滤时，这些字节集就可以被重用，而不需要重新运算整个过滤。
集很“聪明”：他们会增量更新。你索引中添加了新的文档，只有这些新文档需要被添加到已存的字节集中，而不是一遍遍重新计算整个缓存的过滤器。过滤器和整个系统的其他部分一样是实时的，你不需要关心缓存的过期时间。


## 安装
1. 首先需要依赖java7以上版本
2. curl -L -O https://download.elastic.co/elasticsearch/release/org/elasticsearch/distribution/tar/elasticsearch/2.3.3/elasticsearch-2.3.3.tar.gz
tar -xvf elasticsearch-2.3.3.tar.gz
cd elasticsearch-2.3.3/bin
./elasticsearch
3. 可以在启动的时候重写集群和节点的名字
./elasticsearch --cluster.name my_cluster_name --node.name my_node_name
4. 默认，es使用9200端口提供 restapi访问

## 安装配置：
    config/elasticsearch.yml

    network : host : 10.0.0.4
    path: logs: /var/log/elasticsearch 
    data: /var/data/elasticsearch
    cluster: name: <NAME OF YOUR CLUSTER>
    node: name: <NAME OF YOUR NODE>

## 浏览你的集群
es提供了丰富的restapi用于和集群直接通信包括：
1.检查集群，节点，索引，状态和统计
2.管理集群，节点，索引数据和与元数据
3.对索引curd
4.使用高级搜索功能，比如分页，排序，过滤，脚本，聚合和其他很多

## 集群健康：
    curl 'localhost:9200/_cat/health?v'
返回
epoch timestamp cluster status node.total node.data shards pri relo init unassign 
1394735289 14:28:09 elasticsearch green 1 1 0 0 0 0 0

颜色	意义
green	所有主要分片和复制分片都可用
yellow	所有主要分片可用，但不是所有复制分片都可用
red	不是所有的主要分片都可用

## 列举出所有索引：
    curl'localhost:9200/_cat/indices?v'

## 创建一个索引：
    curl -XPUT 'localhost:9200/customer?pretty'

## 创建一个文档：
    curl -XPUT 'localhost:9200/customer/external/1?pretty'-d '{ "name": "John Doe"}'
## 删除一个索引：
    curl -XDELETE 'localhost:9200/customer?pretty'


## 修改你的数据
替代你的文档：
    curl -XPUT 'localhost:9200/customer/external/1?pretty'-d ' 
    { "name": "John Doe" }'

    curl -XPUT 'localhost:9200/customer/external/1?pretty'-d ' 
    { "name": "Jane Doe" }'

如果指定了id，去创建，之前的那个会被覆盖
如果没有指定id，es会随机的生成一个id

## 更新文档：
es并不会真正的更新文档，当我们进行更新操作的时候，es删除原来的文档，
然后添加一个新的文档。

更新文档也可以使用脚本，动态脚本在1.4.3版本默认被禁用
curl -XPOST 'localhost:9200/customer/external/1/_update?pretty'-d ' 
{ "script" : "ctx._source.age += 5" }'

ctx._source指向要被修改的文档

es后续将会提供能力，类似sql中的 UPDATE-WHERE statement

## 删除文档
    curl -XDELETE 'localhost:9200/customer/external/2?pretty'

delete-by-query 插件可以删除满足要求的一类文档

## 批量操作：

    curl -XPOST 'localhost:9200/customer/external/_bulk?pretty' -d ' 
    {"index":{"_id":"1"}} 
    {"name": "John Doe" }
    {"index":{"_id":"2"}} 
    {"name": "Jane Doe" } '
    curl -XPOST 'localhost:9200/customer/external/_bulk?pretty' -d ' 
    {"update":{"_id":"1"}}
    {"doc": { "name": "John Doe becomes Jane Doe" } } 
    {"delete":{"_id":"2"}} '

## 浏览你的数据
### 查询API

took: es搜索使用了多少毫秒
timed_out 是否超时
_shards 告诉我们多少个分片被搜索，和被成功和失败搜索的分片的数量
hits  搜索结果
hits.total  结果数量
hits.hits  搜索结果的列表（默认给出前10个）
_score  评分

### 查询语句：
    Query DSL
    {"query":{"match_all":{}}}

### 规定数目
    curl -XPOST 'localhost:9200/bank/_search?pretty'-d ' 
    { "query": { "match_all": {} }, "size": 1 }'

### 分页
    curl -XPOST 'localhost:9200/bank/_search?pretty'-d '
    {
    "query": { "match_all": {} }, 
    "from": 10, 
    "size": 10 
    }'

### 规定返回指定的field

    curl -XPOST 'localhost:9200/bank/_search?pretty'-d ' 
    { "query": { "match_all": {} },
    "_source": ["account_number",
    "balance"] }'

### match:

    curl -XPOST 'localhost:9200/bank/_search?pretty'-d ' 
    { "query":
        { "match":
            { "account_number": 20
                
            } 
        } 
    }'

bool: 加入布尔逻辑

    curl -XPOST 'localhost:9200/bank/_search?pretty'-d '
    { "query": { "bool": { "must": [ { "match": { "address": "mill" } }, { "match": { "address": "lane" } } ] } } }'
    curl -XPOST 'localhost:9200/bank/_search?pretty'-d ' 
    { "query": { "bool": { "should": [ { "match": { "address": "mill" } }, { "match": {         "address": "lane" } } ] } } }'
    curl -XPOST 'localhost:9200/bank/_search?pretty'-d ' 
    { "query": { "bool": { "must": [ { "match": { "age": "40" } } ], "must_not": [ { "match": { "state": "ID" } } ] } } }'

### filter
使用filter，es不再计算相关性得分，只是严格的按照条件过滤
比如：range

    curl -XPOST 'localhost:9200/bank/_search?pretty'-d '
    { "query": { "bool": { "must": { "match_all": {} }, "filter": { "range": { "balance": {     "gte": 20000, "lte": 30000 } } } } } }'

### Aggregations 
提供能力是分组和提炼你的数据  就像sql里面的GROUP BY
es里你可以查询返回hit和hit的聚合，在一次查询中
而且可以进行多重聚合

安装state分类，然后返回10个状态，按照数量排序
设置size=0，不展示hits，因为我们只关心聚合结果

    SELECT state, COUNT(*) FROM bank GROUP BY state ORDER BY COUNT(*) DESC


平局值  聚合每一种balance的平均值


### suggest
lasticsearch 0.9.0.3终于基于AnalyzingSuggester加上了prefix suggestions ，可直接做搜索提示功能，在0.9.0.1之前版本都是使用外部插件实现的
参考文章 
http://www.nosqldb.cn/1376024289369.html
http://www.cnblogs.com/jiuyuehe/p/3840821.html

# 总结
es是一个简单又复杂的产品，还有很多高级的功能。

# 拓展

## 官方文档：
https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html

## elasticSearch中文社区：
http://elasticsearch.cn/

## elasticsearch 索引优化:
http://itindex.net/detail/52316-elasticsearch-%E7%B4%A2%E5%BC%95-%E4%BC%98%E5%8C%96


## 与其他相似功能产品对比：

http://www.cnblogs.com/chowmin/articles/4629220.html









