---
title: Elasticsearch性能优化
link_title: elkstack-es03
date: 2017-11-27 10:46:05
tags: [Elasticsearch]
categories: ELKstack
thumbnailImage: http://onxkn9cbz.bkt.clouddn.com/07.jpg	
thumbnailImagePosition: left
---
<span/>
<!-- more -->
![](http://onxkn9cbz.bkt.clouddn.com/07.jpg)
<!-- toc -->
# 目录
- 批量提交
- gateway
- 集群状态维护
- 缓存
- 字段数据
- curator
- profiler

# 批量提交

在 CRUD 章节，我们已经知道 ES 的数据写入是如何操作的了。喜欢自己动手的读者可能已经迫不及待的自己写了程序开始往 ES 里写数据做测试。这时候大家会发现：程序的运行速度非常一般，即使 ES 服务运行在本机，一秒钟大概也就能写入几百条数据。

这种速度显然不是 ES 的极限。事实上，每条数据经过一次完整的 HTTP POST 请求和 ES indexing 是一种极大的性能浪费，为此，ES 设计了批量提交方式。在数据读取方面，叫 mget 接口，在数据变更方面，叫 bulk 接口。mget 一般常用于搜索时 ES 节点之间批量获取中间结果集，对于 Elastic Stack 用户，更常见到的是 bulk 接口。

bulk 接口采用一种比较简朴的数据积累格式，示例如下：

```
# curl -XPOST http://127.0.0.1:9200/_bulk -d'
{ "create" : { "_index" : "test", "_type" : "type1"  } }
{ "field1" : "value1" }
{ "delete" : { "_index" : "test", "_type" : "type1" } }
{ "index" : { "_index" : "test", "_type" : "type1", "_id" : "1" } }
{ "field1" : "value2" }
{ "update" : {"_id" : "1", "_type" : "type1", "_index" : "test"} }
{ "doc" : {"field2" : "value2"} }
'
```

格式是，每条 JSON 数据的上面，加一行描述性的元 JSON，指明下一行数据的操作类型，归属索引信息等。

采用这种格式，而不是一般的 JSON 数组格式，是因为接收到 bulk 请求的 ES 节点，就可以不需要做完整的 JSON 数组解析处理，直接按行处理简短的元 JSON，就可以确定下一行数据 JSON 转发给哪个数据节点了。这样，一个固定内存大小的 network buffer 空间，就可以反复使用，又节省了大量 JVM 的 GC。

事实上，产品级的 logstash、rsyslog、spark 都是默认采用 bulk 接口进行数据写入的。对于打算自己写程序的读者，建议采用 Perl 的 `Search::Elasticsearch::Bulk` 或者 Python 的 `elasticsearch.helpers.*` 库。

## bulk size

在配置 bulk 数据的时候，一般需要注意的就是请求体大小(bulk size)。

这里有一点细节上的矛盾，我们知道，HTTP 请求，是可以通过 HTTP 状态码 *100 Continue* 来持续发送数据的。但对于 ES 节点接收 HTTP 请求体的 *Content-Length* 来说，是按照整个大小来计算的。所以，首先，要确保 bulk 数据不要超过 `http.max_content_length` 设置。

那么，是不是尽量让 bulk size 接近这个数值呢？当然不是。

依然是请求体的问题，因为请求体需要全部加载到内存，而 JVM Heap 一共就那么多(按 31GB 算)，过大的请求体，会挤占其他线程池的空间，反而导致写入性能的下降。

再考虑网卡流量，磁盘转速的问题，所以一般来说，建议 bulk 请求体的大小，在 15MB 左右，通过实际测试继续向上探索最合适的设置。

注意：这里说的 15MB 是请求体的字节数，而不是程序里里设置的 bulk size。bulk size 一般指数据的条目数。不要忘了，bulk 请求体中，每条数据还会额外带上一行元 JSON。

以 logstash 默认的 `bulk_size => 5000` 为例，假设单条数据平均大小 200B ，一次 bulk 请求体的大小就是 1.5MB。那么我们可以尝试 `bulk_size => 50000`；而如果单条数据平均大小是 20KB，一次 bulk 大小就是 100MB，显然超标了，需要尝试下调至 `bulk_size => 500`。


# gateway

gateway 是 ES 设计用来长期存储索引数据的接口。一般来说，大家都是用本地磁盘来存储索引数据，即 `gateway.type` 为 `local`。

数据恢复中，有很多策略调整我们已经在之前分片控制小节讲过。除开分片级别的控制以外，gateway 级别也还有一些可优化的地方：

* gateway.recover_after_nodes
  该参数控制集群在达到多少个节点的规模后，才开始数据恢复任务。这样可以避免集群自动发现的初期，分片不全的问题。

* gateway.recover_after_time
  该参数控制集群在达到上条配置设置的节点规模后，再等待多久才开始数据恢复任务。

* gateway.expected_nodes
  该参数设置集群的预期节点总数。在达到这个总数后，即认为集群节点已经完全加载，即可开始数据恢复，不用再等待上条设置的时间。

注意：gateway 中说的节点，仅包括主节点和数据节点，纯粹的 client 节点是不算在内的。如果你有更明确的选择，也可以按需求写：

* gateway.recover_after_data_nodes
* gateway.recover_after_master_nodes
* gateway.expected_data_nodes
* gateway.expected_master_nodes

## 共享存储上的影子副本

虽然 ES 对 gateway 使用 NFS，iscsi 等共享存储的方式极力反对，但是对于较大量级的索引的副本数据，ES 从 1.5 版本开始，还是提供了一种节约成本又不特别影响性能的方式：影子副本(shadow replica)。

首先，需要在集群各节点的 `elasticsearch.yml` 中开启选项：

```
node.enable_custom_paths: true
```

同时，确保各节点使用相同的路径挂载了共享存储，且目录权限为 Elasticsearch 进程用户可读可写。

然后，创建索引：

```
# curl -XPUT 'http://127.0.0.1:9200/my_index' -d '
{
    "index" : {
        "number_of_shards" : 1,
        "number_of_replicas" : 4,
        "data_path": "/var/data/my_index",
        "shadow_replicas": true
    }
}'
```

针对 shadow replicas ，ES 节点不会做实际的索引操作，而是单纯的每次 flush 时，把 segment 内容 fsync 到共享存储磁盘上。然后 refresh 让其他节点能够搜索该 segment 内容。

如果你已经决定把数据放到共享存储上了，采用 shadow replicas 还是有一些好处的：

1. 可以帮助你节省一部分不必要的多副本分片的数据写入压力；
2. 在节点出现异常，需要在其他节点上恢复副本数据的时候，可以避免不必要的网络数据拷贝。

但是请注意：主分片节点还是要承担一个副本的写入过程，并不像 Lucene 的 FileReplicator 那样通过复制文件完成，所以达不到完全节省 CPU 的效果。

shadow replicas 只是一个在某些特定环境下有用的方式。在资源允许的情况下，还是应该使用 local gateway。而另外采用 snapshot 接口来完成数据长期备份到 HDFS 或其他共享存储的需要。


# 集群状态维护

我们都知道，ES 中的 master 跟一般 MySQL、Hadoop 的 master 是不一样的。它即不是写入流量的唯一入口，也不是所有数据的元信息的存放地点。所以，一般来说，ES 的 master 节点负载很轻，集群性能是可以近似认为随着 data 节点的扩展线性提升的。

但是，上面这句话并不是完全正确的。

ES 中有一件事情是只有 master 节点能管理的，这就是集群状态(cluster state)。

集群状态中包括以下信息：

* 集群层面的设置
* 集群内有哪些节点
* 各索引的设置，映射，分析器和别名等
* 索引内各分片所在的节点位置

这些信息在集群的任意节点上都存放着，你也可以通过 `/_cluster/state` 接口直接读取到其内容。注意这最后一项信息，之前我们已经讲过 ES 怎么通过简单地取余知道一条数据放在哪个分片里，加上现在集群状态里又记载了分片在哪个节点上，那么，整个集群里，任意节点都可以知道一条数据在哪个节点上存储了。所以，数据读写才可以发送给集群里任意节点。

至于修改，则只能由 master 节点完成！显然，集群状态里大部分内容是极少变动的，唯独有一样除外——索引的映射。因为 ES 的 schema-less 特性，我们可以任意写入 JSON 数据，所以索引中随时可能增加新的字段。这个时候，负责容纳这条数据的主分片所在的节点，会暂停写入操作，将字段的映射结果传递给 master 节点；master 节点合并这段修改到集群状态里，发送新版本的集群状态到集群的所有节点上。然后写入操作才会继续。一般来说，这个操作是在一二十毫秒内就可以完成，影响也不大。

但是也有一些情况会是例外。

## 批量新索引创建

在较大规模的 Elastic Stack 应用场景中，这是比较常见的一个情况。因为 Elastic Stack 建议采用日期时间作为索引的划分方式，所以定时(一般是每天)，会统一产生一批新的索引。而前面已经讲过，ES 的集群状态每次更新都是阻塞式的发布到全部节点上以后，节点才能继续后续处理。

这就意味着，如果在集群负载较高的时候，批量新建新索引，可能会有一个显著的阻塞时间，无法写入任何数据。要等到全部节点同步完成集群状态以后，数据写入才能恢复。

不巧的是，中国使用的是北京时间，UTC +0800。也就是说，默认的 Elastic Stack 新建索引时间是在早上 8 点。这个时间点一般日志写入量已经上涨到一定水平了(当然，晚上 0 点的量其实也不低)。

对此，可以通过定时任务，每天在最低谷的早上三四点，提前通过 POST mapping 的方式，创建好之后几天的索引。就可以避免这个问题了。

**如果你的日志是比较严重的非结构化数据，这个问题在 2.0 版本后会变得更加严重。** Elasticsearch 从 2.0 版本开始，对 mapping 更新做了重构。为了防止字段类型冲突和减少 master 定期下发全量 cluster state 导致的大流量压力，新的实现和旧实现的区别在：

* 过去：每次 bulk 请求，本地生成索引后，将更新的 mapping，按照 `_type` 为单位构成 mapping 更新请求发给 master；
* 现在：每次 bulk 请求，遍历每条数据，将每条数据要更新的 mapping，都单独发给 master，等到 master 通知完全集群，本地才能生成这一条数据的索引。

也就是说，一旦你日志中字段数量较多，在新创建索引的一段时间内，可能长达几十分钟一直被反复锁死！

## 过多字段持续更新

这是另一种常见的滥用。在使用 Elastic Stack 处理访问日志时，为了查询更方便，可能会采用 logstash-filter-kv 插件，将访问日志中的每个 URL 参数，都切分成单独的字段。比如一个 "/index.do?uid=1234567890&action=payload" 的 URL 会被转换成如下 JSON：

```
  "urlpath" : "/index.do",
  "urlargs" : {
    "uid" : "1234567890",
    "action" : "payload",
    ...
  }
```

但是，因为集群状态是存在所有节点的内存里的，一旦 URL 参数过多，ES 节点的内存就被大量用于存储字段映射内容。这是一个极大的浪费。如果碰上 URL 参数的键内容本身一直在变动，直接撑爆 ES 内存都是有可能的！

*以上是真实发生的事件，开发人员莫名的选择将一个 UUID 结果作为 key 放在 URL 参数里。直接导致 ES 集群 master 节点全部 OOM。*

如果你在 ES 日志中一直看到有新的 `updating mapping [logstash-2015.06.01]` 字样出现的话，请郑重考虑一下自己是不是用的上如此细分的字段列表吧。

好，三秒钟过去，如果你确定一定以及肯定还要这么做，下面是一个变通的解决办法。

### nested object

用 nested object 来存放 URL 参数的方法稍微复杂，但还可以接受。单从 JSON 数据层面看，新方式的数据结构如下：

```
  "urlargs": [
    { "key": "uid", "value": "1234567890" },
    { "key": "action", "value": "payload" },
    ...
  ]
```

没错，看起来就是一个数组。但是 JSON 数组在 ES 里是有两种处理方式的。

如果直接写入数组，ES 在实际索引过程中，会把所有内容都平铺开，变成 **Arrays of Inner Objects**。整条数据实际类似这样的结构：

```
{
  "urlpath" : ["/index.do"],
  "urlargs.key" : ["uid", "action", ...],
  "urlargs.value" : ["1234567890", "payload", ...]
```

这种方式最大的问题是，当你采用 `urlargs.key:"uid" AND urlargs.value:"0987654321"` 语句意图搜索一个 uid=0987654321 的请求时，实际是整个 URL 参数中任意一处 value 为 0987654321 的，都会命中。

要想达到正确搜索的目的，需要在写入数据之前，指定 urlargs 字段的映射类型为 nested object。命令如下：

```
curl -XPOST http://127.0.0.1:9200/logstash-2015.06.01/_mapping -d '{
  "accesslog" : {
    "properties" : {
      "urlargs" : {
        "type" : "nested",
        "properties" : {
            "key" : { "type" : "string", "index" : "not_analyzed", "doc_values" : true },
            "value" : { "type" : "string", "index" : "not_analyzed", "doc_values" : true }
        }
      }
    }
  } 
}'
```

这样，数据实际是类似这样的结构：

```
{
  "urlpath" : ["/index.do"],
},
{
  "urlargs.key" : ["uid"],
  "urlargs.value" : ["1234567890"],
},
{
  "urlargs.key" : ["action"],
  "urlargs.value" : ["payload"],
}
```

当然，nested object 节省字段映射的优势对应的是它在使用的复杂。Query 和 Aggs 都必须使用专门的 nested query 和 nested aggs 才能正确读取到它。

nested query 语法如下：

```
curl -XPOST http://127.0.0.1:9200/logstash-2015.06.01/accesslog/_search -d '
{
  "query": {
    "bool": {
      "must": [
        { "match": { "urlpath" : "/index.do" }}, 
        {
          "nested": {
            "path": "urlargs", 
            "query": {
              "bool": {
                "must": [ 
                  { "match": { "urlargs.key": "uid" }},
                  { "match": { "urlargs.value": "1234567890" }}
                ]
        }}}}
      ]
}}}'
```

nested aggs 语法如下：

```
curl -XPOST http://127.0.0.1:9200/logstash-2015.06.01/accesslog/_search -d '
{
  "aggs": {
    "topnuid": {
      "nested": {
        "path": "urlargs"
      },
      "aggs": {
        "uid": {
          "filter": {
            "term": {
              "urlargs.key": "uid",
            }
          },
          "aggs": {
            "topn": {
              "terms": { 
                "field": "urlargs.value"
              }
            }
          }
        }
      }
    }
  }
}'
```


# 缓存

ES 内针对不同阶段，设计有不同的缓存。以此提升数据检索时的响应性能。主要包括节点层面的 filter cache 和分片层面的 request cache。下面分别讲述。

## filter cache

ES 的 query DSL 在 2.0 版本之前分为 query 和 filter 两种，很多检索语法，是同时存在 query 和 filter 里的。比如最常用的 term、prefix、range 等。怎么选择是使用 query 还是 filter 成为很多用户头疼的难题。于是从 2.0 版本开始，ES 干脆合并了 filter 统一归为 query。但是具体的检索语法本身，依然有 query 和 filter 上下文的区别。ES 依靠这个上下文判断，来自动决定是否启用 filter cache。

query 跟 filter 上下文的区别，简单来说：

* query 是要相关性评分的，filter 不要；
* query 结果无法缓存，filter 可以。

所以，选择也就出来了：

* 全文搜索、评分排序，使用 query；
* 是非过滤，精确匹配，使用 filter。

不过我们要怎么写，才能让 ES 正确判断呢？看下面这个请求：

```
# curl -XGET http://127.0.0.1:9200/_search -d '
{
    "query": {
        "bool": {
            "must_not": [
                { "match": { "title": "Search" } }
            ],
            "must": [
                { "match": { "content": "Elasticsearch" } }
            ],
            "filter": [
                { "term":  { "status": "published" } },
                { "range": { "publish_date": { "gte": "2015-01-01" } } }
            ]
        }
    }
}'
```

在这个请求中，

1. ES 先看到一个 **query**，那么进入 query 上下文。
2. 然后在 **bool** 里看到一个 **must_not**，那么改进入 filter 上下文，这个有关 title 字段的查询不参与评分。
3. 然后接着是一个 **must** 的 **match**，这个又属于 query 上下文，这个有关 content 字段的查询会影响评分。
4. 最后碰到 **filter**，还属于 filter 上下文，这个有关 status 和 publish_date 字段的查询不参与评分。

需要注意的是，filter cache 是节点层面的缓存设置，每个节点上所有数据在响应请求时，是共用一个缓存空间的。当空间用满，按照 LRU 策略淘汰掉最冷的数据。

可以用 `indices.cache.filter.size` 配置来设置这个缓存空间的大小，默认是 JVM 堆的 10%，也可以设置一个绝对值。注意这是一个静态值，必须在 `elasticsearch.yml` 中提前配置。

## shard request cache

ES 还有另一个分片层面的缓存，叫 shard request cache。5.0 之前的版本中，request cache 的用途并不大，因为 query cache 要起作用，还有几个先决条件：

1. 分片数据不再变动，也就是对当天的索引是无效的(如果 `refresh_interval` 很大，那么在这个间隔内倒也算有效)；
2. 使用了 `"now"` 语法的请求无法被缓存，因为这个是要即时计算的；
3. 缓存的键是请求的整个 JSON 字符串，整个字符串发生任何字节变动，缓存都无效。

以 Elastic Stack 场景来说，Kibana 里几乎所有的请求，都是有 `@timestamp` 作为过滤条件的，而且大多数是以*最近 N 小时/分钟*这样的选项，也就是说，页面每次刷新，发出的请求 JSON 里的时间过滤部分都是在变动的。query cache 在处理 Kibana 发出的请求时，完全无用。

而 5.0 版本的一大特性，叫 instant aggregation。解决了这个先决条件的一大阻碍。

在之前的版本，Elasticsearch 接收到请求之后，直接把请求原样转发给各分片，由各分片所在的节点自行完成请求的解析，进行实际的搜索操作。所以缓存的键是原始 JSON 串。

而 5.0 的重构后，接收到请求的节点先把请求的解析做完，发送到各节点的是统一拆分修改好的请求，这样就不再担心 JSON 串多个空格啥的了。

其次，上面说的『拆分修改』是怎么回事呢？

比如，我们在 Kibana 里搜索一个最近 7 天(`@timestamp:["now-7d" TO "now"]`)的数据，ES 就可以根据按天索引的判断，知道从 6 天前到昨天这 5 个索引是肯定全覆盖的。那么这个横跨 7 天的 `date range` query 就变成了 5 个 `match_all` query 加 2 个短时间的 `date_range` query。

现在你的仪表盘过 5 分钟自动刷新一次，再提交上来一次最近 7 天的请求，中间这 5 个 `match_all` 就完全一样了，直接从 request cache 返回即可，需要重新请求的，只有两头真正在变动的 `date_range` 了。

*注1：`match_all` 不用遍历倒排索引，比直接查询 `@timestamp:*` 要快很多。*
*注2：判断覆盖修改为 `match_all` 并不是真的按照索引名称，而是 ES 从 2.x 开始提供的 `field_stats` 接口可以直接获取到 `@timestamp` 在本索引内的 max/min 值。当然从概念上如此理解也是可以接受的。*

### field_stats 接口

```
curl -XGET "http://localhost:9200/logstash-2016.11.25/_field_stats?fields=timestamp"
```

响应结果如下：

```
{
    "_shards": {
        "total": 1,
        "successful": 1,
        "failed": 0
    },
    "indices": {
        "logstash-2016.11.25": {
            "fields": {
                "timestamp": {
                    "max_doc": 1326564,
                    "doc_count": 564633,
                    "density": 42,
                    "sum_doc_freq": 2258532,
                    "sum_total_term_freq": -1,
                    "min_value": "2008-08-01T16:37:51.513Z",
                    "max_value": "2013-06-02T03:23:11.593Z",
                    "is_searchable": "true",
                    "is_aggregatable": "true"
                }
            }
        }
    }
}
```

和 filter cache 一样，request cache 的大小也是以节点级别控制的，配置项名为 `indices.requests.cache.size`，其默认值为 `1%`。


# 字段数据

字段数据(fielddata)，在 Lucene 中又叫 uninverted index。我们都知道，搜索引擎会使用倒排索引(inverted index)来映射单词到文档的 ID 号。而同时，为了提供对文档内容的聚合，Lucene 还可以在运行时将每个字段的单词以字典序排成另一个 uninverted index，可以大大加速计算性能。

作为一个加速性能的方式，fielddata 当然是被全部加载在内存的时候最为有效。这也是 ES 默认的运行设置。但是，内存是有限的，所以 ES 同时也需要提供对 fielddata 内存的限额方式：

* indices.fielddata.cache.size
  节点用于 fielddata 的最大内存，如果 fielddata 达到该阈值，就会把旧数据交换出去。该参数可以设置百分比或者绝对值。默认设置是不限制，所以强烈建议设置该值，比如 `10%`。
* indices.fielddata.cache.expire
  进入 fielddata 内存中的数据多久自动过期。注意，因为 ES 的 fielddata 本身是一种数据结构，而不是简单的缓存，所以过期删除 fielddata 是一个非常消耗资源的操作。ES 官方在文档中特意说明，这个参数绝对绝对**不要**设置！

### Circuit Breaker

Elasticsearch 在 total，fielddata，request 三个层面上都设计有 circuit breaker 以保护进程不至于发生 OOM 事件。在 fielddata 层面，其设置为：

* indices.breaker.fielddata.limit
  默认是 JVM 堆内存大小的 60%。注意，为了让设置正常发挥作用，如果之前设置过 `indices.fielddata.cache.size` 的，一定要确保 `indices.breaker.fielddata.limit` 的值大于 `indices.fielddata.cache.size` 的值。否则的话，fielddata 大小一到 limit 阈值就报错，就永远道不了 size 阈值，无法触发对旧数据的交换任务了。

## doc values

但是相比较集群庞大的数据量，内存本身是远远不够的。为了解决这个问题，ES 引入了另一个特性，可以对精确索引的字段，指定 fielddata 的存储方式。这个配置项叫：`doc_values`。

所谓 `doc_values`，其实就是在 ES 将数据写入索引的时候，提前生成好 fielddata 内容，并记录到磁盘上。因为 fielddata 数据是顺序读写的，所以即使在磁盘上，通过文件系统层的缓存，也可以获得相当不错的性能。

注意：因为 `doc_values` 是在数据写入时即生成内容，所以，它只能应用在精准索引的字段上，因为索引进程没法知道后续会有什么分词器生成的结果。

由于在 Elastic Stack 场景中，`doc_values` 的使用极其频繁，到 Elasticsearch 5.0 以后，这两者的区别被彻底强化成两个不同字段类型：`text` 和 `keyword`。

```
"myfieldname": {
    "type": "text"
}
```

等同于过去的：

```
    "myfieldname": {
        "type": "string",
        "fielddata": false
    }
```

而

```
"myfieldname": {
    "type": "keyword"
}
```

等同于过去的：

```
    "myfieldname": {
        "type": "string",
        "index": "not_analyzed",
        "doc_values": true
    }
```

也就是说，以后的用户，已经不太需要在意 fielddata 的问题了。不过依然有少数情况，你会需要对分词字段做聚合统计的话，你可以在自己接受范围内，开启这个特性：

```
{
    "mappings": {
        "my_type": {
            "properties": {
                "message": {
                    "type": "text",
                    "fielddata": true,
                    "fielddata_frequency_filter": {
                        "min": 0.1,
                        "max": 1.0,
                        "min_segment_size": 500
                    }
                }
            }
        }
    }
}
```

你可以看到在上面加了一段 `fielddata_frequency_filter` 配置，这个配置是 segment 级别的。上面示例的意思是：只有这个 segment 里的文档数量超过 500 个，而且含有该字段的文档数量占该 segment 里的文档数量比例超过 10% 时，才加载这个 segment 的 fielddata。

下面是一个可能有用的对分词字段做聚合的示例：

```
curl -XPOST 'http://localhost:9200/logstash-2016.07.18/logs/_search?pretty&terminate_after=10000&size=0' -d '
{
    "aggs": {
        "group": {
            "terms": {
                "field": "punct"
            },
            "aggs": {
                "keyword": {
                    "significant_terms": {
                        "size": 2,
                        "field": "message"
                    },
                    "aggs": {
                        "hit": {
                            "top_hits": {
                                "_source": {
                                    "include": [ "message" ]
                                },
                                "size":1
                            }
                        }
                    }
                }
            }
        }
    }
}'
```

这个示例可以对经过了 `logstash-filter-punct` 插件处理的数据，获取每种 punct 类型日志的关键词和对应的代表性日志原文。其效果类似 Splunk 的事件模式功能：

![](http://chenlinux.com/images/uploads/splunk-event-pattern.png)


# curator

如果经过之前章节的一系列优化之后，数据确实超过了集群能承载的能力，除了拆分集群以外，最后就只剩下一个办法了：清除废旧索引。

为了更加方便的做清除数据，合并 segment，备份恢复等管理任务，Elasticsearch 在提供相关 API 的同时，另外准备了一个命令行工具，叫 curator 。curator 是 Python 程序，可以直接通过 pypi 库安装：

```
pip install elasticsearch-curator
```

*注意，是 elasticsearch-curator 不是 curator。PyPi 原先就有另一个项目叫这个名字*

## 参数介绍

和 Elastic Stack 里其他组件一样，curator 也是被 Elastic.co 收购的原开源社区周边。收编之后同样进行了一次重构，命令行参数从单字母风格改成了长单词风格。新版本的 curator 命令可用参数如下：

> Usage: curator [OPTIONS] COMMAND [ARGS]...

Options 包括:

  --host TEXT        Elasticsearch host.
  --url_prefix TEXT  Elasticsearch http url prefix.
  --port INTEGER     Elasticsearch port.
  --use_ssl          Connect to Elasticsearch through SSL.
  --http_auth TEXT   Use Basic Authentication ex: user:pass
  --timeout INTEGER  Connection timeout in seconds.
  --master-only      Only operate on elected master node.
  --dry-run          Do not perform any changes.
  --debug            Debug mode
  --loglevel TEXT    Log level
  --logfile TEXT     log file
  --logformat TEXT   Log output format [default|logstash].
  --version          Show the version and exit.
  --help             Show this message and exit.

Commands 包括:
  alias       Index Aliasing
  allocation  Index Allocation
  bloom       Disable bloom filter cache
  close       Close indices
  delete      Delete indices or snapshots
  open        Open indices
  optimize    Optimize Indices
  replicas    Replica Count Per-shard
  show        Show indices or snapshots
  snapshot    Take snapshots of indices (Backup)

针对具体的 Command，还可以继续使用 `--help` 查看该子命令的帮助。比如查看 *close* 子命令的帮助，输入 `curator close --help`，结果如下：

```
Usage: curator close [OPTIONS] COMMAND [ARGS]...

  Close indices

Options:
  --help  Show this message and exit.

Commands:
  indices  Index selection.
```

## 常用示例

在使用 1.4.0 以上版本的 Elasticsearch 前提下，curator 曾经主要的一个子命令 `bloom` 已经不再需要使用。所以，目前最常用的三个子命令，分别是 `close`, `delete` 和 `optimize`，示例如下：

```
curator --timeout 36000 --host 10.0.0.100 delete indices --older-than 5 --time-unit days --timestring '%Y.%m.%d' --prefix logstash-mweibo-nginx-
curator --timeout 36000 --host 10.0.0.100 delete indices --older-than 10 --time-unit days --timestring '%Y.%m.%d' --prefix logstash-mweibo-client- --exclude 'logstash-mweibo-client-2015.05.11'
curator --timeout 36000 --host 10.0.0.100 delete indices --older-than 30 --time-unit days --timestring '%Y.%m.%d' --regex '^logstash-mweibo-\d+'
curator --timeout 36000 --host 10.0.0.100 close indices --older-than 7 --time-unit days --timestring '%Y.%m.%d' --prefix logstash-
curator --timeout 36000 --host 10.0.0.100 optimize --max_num_segments 1 indices --older-than 1 --newer-than 7 --time-unit days --timestring '%Y.%m.%d' --prefix logstash-
```

这一顿任务，结果是：

*logstash-mweibo-nginx-yyyy.mm.dd* 索引保存最近 5 天，*logstash-mweibo-client-yyyy.mm.dd* 保存最近 10 天，*logstash-mweibo-yyyy.mm.dd* 索引保存最近 30 天；且所有七天前的 *logstash-\** 索引都暂时关闭不用；最后对所有非当日日志做 segment 合并优化。

# profiler

profiler 是 Elasticsearch 5.0 的一个新接口。通过这个功能，可以看到一个搜索聚合请求，是如何拆分成底层的 Lucene 请求，并且显示每部分的耗时情况。

启用 profiler 的方式很简单，直接在请求里加一行即可：

```
curl -XPOST 'http://localhost:9200/_search' -d '{
    "profile": true,
    "query": { ... },
    "aggs": { ... }
}'
```

可以看到其中对 query 和 aggs 部分的返回是不太一样的。

## query

query 部分包括 collectors、rewrite 和 query 部分。对复杂 query，profiler 会拆分 query 成多个基础的 TermQuery，然后每个 TermQuery 再显示各自的分阶段耗时如下：

```
"breakdown": {
    "score": 51306,
    "score_count": 4,
    "build_scorer": 2935582,
    "build_scorer_count": 1,
    "match": 0,
    "match_count": 0,
    "create_weight": 919297,
    "create_weight_count": 1,
    "next_doc": 53876,
    "next_doc_count": 5,
    "advance": 0,
    "advance_count": 0
}
```



## aggs

```
        "time": "1124.864392ms",
        "breakdown": {
            "reduce": 0,
            "reduce_count": 0,
            "build_aggregation": 1394,
            "build_aggregation_count": 150,
            "initialise": 2883,
            "initialize_count": 150,
            "collect": 1124860115,
            "collect_count": 900
        }
```

我们可以很明显的看到聚合统计在初始化阶段、收集阶段、构建阶段、汇总阶段分别花了多少时间，遍历了多少数据。

*注意其中 reduce 阶段还没实现完毕，所有都是 0。因为目前 profiler 只能在 shard 级别上做统计。*

collect 阶段的耗时，有助于我们调整对应 aggs 的 `collect_mode` 参数选择。目前 Elasticsearch 支持 `breadth_first` 和 `depth_first` 两种方式。

initialise 阶段的耗时，有助于我们调整对应 aggs 的 `execution_hint` 参数选择。目前 Elasticsearch 支持 `map`、`global_ordinals_low_cardinality`、`global_ordinals` 和 `global_ordinals_hash` 四种选择。在计算离散度比较大的字段统计值时，适当调整该参数，有益于节省内存和提高计算速度。

*对高离散度字段值统计性能很关注的读者，可以关注 <https://github.com/elastic/elasticsearch/pull/21626> 这条记录的进展。*

（本文完）

文本整理自《ELKstack权威指南》
