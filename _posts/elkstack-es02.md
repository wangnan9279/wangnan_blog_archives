---
title: Elasticsearch数据接口用例
link_title: elkstack-es02
date: 2018-10-16 13:56:15
tags: [Elasticsearch]
categories: ELKstack
thumbnailImage: https://upload-images.jianshu.io/upload_images/79431-4549186abf87e6f8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/475/format/webp
thumbnailImagePosition: left
---
<span/>
<!-- more -->
![](https://upload-images.jianshu.io/upload_images/79431-4549186abf87e6f8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/475/format/webp)
<!-- toc -->
# 目录
- 增删改查
- 搜索请求
- script
- reindex

# 增删改查

增删改查是数据库的基础操作方法。ES 虽然不是数据库，但是很多场合下，都被人们当做一个文档型 NoSQL 数据库在使用，原因自然是因为在接口和分布式架构层面的相似性。虽然在 Elastic Stack 场景下，数据的写入和查询，分别由 Logstash 和 Kibana 代劳，作为测试、调研和排错时的基本功，还是需要了解一下 ES 的增删改查用法的。

## 数据写入

ES 的一大特点，就是全 RESTful 接口处理 JSON 请求。所以，数据写入非常简单：

```
# curl -XPOST http://127.0.0.1:9200/logstash-2015.06.21/testlog -d '{
    "date" : "1434966686000",
    "user" : "chenlin7",
    "mesg" : "first message into Elasticsearch"
}'
```

命令返回响应结果为：

```
{"_index":"logstash-2015.06.21","_type":"testlog","_id":"AU4ew3h2nBE6n0qcyVJK","_version":1,"created":true}
```

## 数据获取

可以看到，在数据写入的时候，会返回该数据的 `_id`。这就是后续用来获取数据的关键：

```
# curl -XGET http://127.0.0.1:9200/logstash-2015.06.21/testlog/AU4ew3h2nBE6n0qcyVJK
```

命令返回响应结果为：

```
{"_index":"logstash-2015.06.21","_type":"testlog","_id":"AU4ew3h2nBE6n0qcyVJK","_version":1,"found":true,"_source":{
    "date" : "1434966686000",
    "user" : "chenlin7",
    "mesg" : "first message into Elasticsearch"
}}
```

这个 `_source` 里的内容，正是之前写入的数据。

如果觉得这个返回看起来有点太过麻烦，可以使用 `curl -XGET http://127.0.0.1:9200/logstash-2015.06.21/testlog/AU4ew3h2nBE6n0qcyVJK/_source` 来指明只获取源数据部分。

更进一步的，如果你只想看数据中的一部分字段内容，可以使用 `curl -XGET http://127.0.0.1:9200/logstash-2015.06.21/testlog/AU4ew3h2nBE6n0qcyVJK?fields=user,mesg` 来指明获取字段，结果如下：

```
{"_index":"logstash-2015.06.21","_type":"testlog","_id":"AU4ew3h2nBE6n0qcyVJK","_version":1,"found":true,"fields":{"user":["chenlin7"],"mesg":["first message into Elasticsearch"]}}
```

## 数据删除

要删除数据，修改发送的 HTTP 请求方法为 DELETE 即可：

```
# curl -XDELETE http://127.0.0.1:9200/logstash-2015.06.21/testlog/AU4ew3h2nBE6n0qcyVJK
```

删除不单针对单条数据，还可以删除整个整个索引。甚至可以用通配符。

```
# curl -XDELETE http://127.0.0.1:9200/logstash-2015.06.0*
```

在 Elasticsearch 2.x 之前，可以通过查询语句删除，也可以删除某个 `_type` 内的数据。现在都已经不再内置支持，改为 `Delete by Query` 插件。因为这种方式本身对性能影响较大！

## 数据更新

已经写过的数据，同样还是可以修改的。有两种办法，一种是全量提交，即指明 `_id` 再发送一次写入请求。

```
# curl -XPOST http://127.0.0.1:9200/logstash-2015.06.21/testlog/AU4ew3h2nBE6n0qcyVJK -d '{
    "date" : "1434966686000",
    "user" : "chenlin7",
    "mesg" " "first message into Elasticsearch but version 2"
}'
```

另一种是局部更新，使用 `/_update` 接口：

```
# curl -XPOST 'http://127.0.0.1:9200/logstash-2015.06.21/testlog/AU4ew3h2nBE6n0qcyVJK/_update' -d '{
    "doc" : {
        "user" : "someone"
    }
}'
```

或者

```
# curl -XPOST 'http://127.0.0.1:9200/logstash-2015.06.21/testlog/AU4ew3h2nBE6n0qcyVJK/_update' -d '{
    "script" : "ctx._source.user = \"someone\""
}'
```


# 搜索请求

上节介绍的，都是针对单条数据的操作。在 ES 环境中，更多的是搜索和聚合请求。在 5.0 之前版本中，数据获取和数据搜索甚至有极大的区别：刚写入的数据，可以通过 translog 立刻获取；但是却要等到 refresh 成为一个 segment 后，才能被搜索到。从 5.0 版本开始，Elasticsearch 稍作了改动，不再维护 doc-id 到 translog offset 的映射关系，一旦 GET 请求到这个还不能搜到的数据，就强制 refresh 出来 segment，这样就可以搜索了。这个改动降低了数据获取的性能，但是节省了不少内存，减少了 young GC 次数，对写入性能的提升是很有好处的。

本节介绍 ES 的搜索语法。

## 全文搜索

ES 对搜索请求，有简易语法和完整语法两种方式。简易语法作为以后在 Kibana 上最常用的方式，一定是需要学会的。而在命令行里，我们可以通过最简单的方式来做到。还是上节输入的数据：

```
# curl -XGET http://127.0.0.1:9200/logstash-2015.06.21/testlog/_search?q=first
```

可以看到返回结果：

```
{"took":240,"timed_out":false,"_shards":{"total":27,"successful":27,"failed":0},"hits":{"total":1,"max_score":0.11506981,"hits":[{"_index":"logstash-2015.06.21","_type":"testlog","_id":"AU4ew3h2nBE6n0qcyVJK","_score":0.11506981,"_source":{
    "date" : "1434966686000",
    "user" : "chenlin7",
    "mesg" : "first message into Elasticsearch"
}}]}}
```

还可以用下面语句搜索，结果是一样的。

```
# curl -XGET http://127.0.0.1:9200/logstash-2015.06.21/testlog/_search?q=user:"chenlin7"
```

### querystring 语法

上例中，`?q=`后面写的，就是 querystring 语法。鉴于这部分内容会在 Kibana 上经常使用，这里详细解析一下语法：

* 全文检索：直接写搜索的单词，如上例中的 `first`；
* 单字段的全文检索：在搜索单词之前加上字段名和冒号，比如如果知道单词 first 肯定出现在 mesg 字段，可以写作 `mesg:first`；
* 单字段的精确检索：在搜索单词前后加双引号，比如 `user:"chenlin7"`；
* 多个检索条件的组合：可以使用 `NOT`, `AND` 和 `OR` 来组合检索，注意必须是大写。比如 `user:("chenlin7" OR "chenlin") AND NOT mesg:first`；
* 字段是否存在：`_exists_:user` 表示要求 user 字段存在，`_missing_:user` 表示要求 user 字段不存在；
* 通配符：用 `?` 表示单字母，`*` 表示任意个字母。比如 `fir?t mess*`；
* 正则：需要比通配符更复杂一点的表达式，可以使用正则。比如 `mesg:/mes{2}ages?/`。注意 ES 中正则性能很差，而且支持的功能也不是特别强大，尽量不要使用。ES 支持的正则语法见：<https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-regexp-query.html#regexp-syntax>；
* 近似搜索：用 `~` 表示搜索单词可能有一两个字母写的不对，请 ES 按照相似度返回结果。比如 `frist~`；
* 范围搜索：对数值和时间，ES 都可以使用范围搜索，比如：`rtt:>300`，`date:["now-6h" TO "now"}` 等。其中，`[]` 表示端点数值包含在范围内，`{}` 表示端点数值不包含在范围内；

### 完整语法

ES 支持各种类型的检索请求，除了可以用 querystring 语法表达的以外，还有很多其他类型，具体列表和示例可参见：<https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-queries.html>。

作为最简单和常用的示例，这里展示一下 term query 的写法，相当于 querystring 语法中的 `user:"chenlin7"`：

```
# curl -XGET http://127.0.0.1:9200/_search -d '
{
  "query": {
    "term": {
      "user": "chenlin7" 
    }
  }
}'
```

## 聚合请求

在检索范围确定之后，ES 还支持对结果集做聚合查询，返回更直接的聚合统计结果。在 ES 1.0 版本之前，这个接口叫 Facet，1.0 版本之后，这个接口改为 Aggregation。

Kibana 分别在 v3 中使用 Facet，v4 中使用 Aggregation。不过总的来说，Aggregation 是 Facet 接口的强化升级版本，我们直接了解 Aggregation 即可。本书后续章节也会介绍如何在 Kibana 的 v3 版本中使用 aggregation 接口做二次开发。

### 堆叠聚合示例

在 Elasticsearch 1.x 系列中，aggregation 分为 bucket 和 metric 两种，分别用作词元划分和数值计算。而其中的 bucket aggregation，还支持在自身结果集的基础上，叠加新的 aggregation。这就是 aggregation 比 facet 最领先的地方。比如实现一个时序百分比统计，在 facet 接口就无法直接完成，而在 aggregation 接口就很简单了：

```
# curl -XPOST 'http://127.0.0.1:9200/logstash-2015.06.22/_search?size=0&pretty' -d'{
    "aggs" : {
        "percentile_over_time" : {
            "date_histogram" : {
                "field"    : "@timestamp",
                "interval" : "1h"
            },
            "aggs" : {
                "percentile_one_time" : {
                    "percentiles" : {
                        "field" : "requesttime"
                    }
                }
            }
        }
    }
}'
```

得到结果如下：

```
{
  "took" : 151595,
  "timed_out" : false,
  "_shards" : {
    "total" : 81,
    "successful" : 81,
    "failed" : 0
  },
  "hits" : {
    "total" : 3307142043,
    "max_score" : 1.0,
    "hits" : [ ]
  },
  "aggregations" : {
    "percentile_over_time" : {
      "buckets" : [ {
        "key_as_string" : "22/Jun/2015:22:00:00 +0000",
        "key" : 1435010400000,
        "doc_count" : 459273981,
        "percentile_one_time" : {
          "values" : {
            "1.0" : 0.004,
            "5.0" : 0.006,
            "25.0" : 0.023,
            "50.0" : 0.035,
            "75.0" : 0.08774675719725569,
            "95.0" : 0.25732934416125663,
            "99.0" : 0.7508899754871812
          }
        }
      }, {
        "key_as_string" : "23/Jun/2015:00:00:00 +0000",
        "key" : 1435017600000,
        "doc_count" : 768620219,
        "percentile_one_time" : {
          "values" : {
            "1.0" : 0.004,
            "5.0" : 0.007000000000000001,
            "25.0" : 0.025,
            "50.0" : 0.03987809503972864,
            "75.0" : 0.10297843567746187,
            "95.0" : 0.30047269327062875,
            "99.0" : 1.015495933753329
          }
        }
      }, {
        "key_as_string" : "23/Jun/2015:02:00:00 +0000",
        "key" : 1435024800000,
        "doc_count" : 849467060,
        "percentile_one_time" : {
          "values" : {
            "1.0" : 0.004,
            "5.0" : 0.008,
            "25.0" : 0.027000000000000003,
            "50.0" : 0.0439999899006102,
            "75.0" : 0.1160416197625958,
            "95.0" : 0.3383140614483838,
            "99.0" : 1.0275839684542212
          }
        }
      } ]
    }
  }
}
```

### 管道聚合示例

在 Elasticsearch 2.x 中，新增了 pipeline aggregation 类型。可以在已有 aggregation 返回的数组数据之后，再对这组数值做一次运算。最常见的，就是对时序数据求移动平均值。比如对响应时间做周期为 7，移动窗口为 30，alpha, beta, gamma 参数均为 0.5 的 holt-winters 季节性预测 2 个未来值的请求如下：

```
{
    "aggs" : {
        "my_date_histo" : {
            "date_histogram" : {
                 "field" : "@timestamp",
                 "interval" : "1h"
            },
            "aggs" : {
                "avgtime" : {
                    "avg" : { "field" : "requesttime" }
                },
                "the_movavg" : {
                    "moving_avg" : {
                        "buckets_path" : "avgtime",
                        "window" : 30,
                        "model" : "holt_winters",
                        "predict" : 2,
                        "settings" : {
                            "type" : "mult",
                            "alpha" : 0.5,
                            "beta" : 0.5,
                            "gamma" : 0.5,
                            "period" : 7,
                            "pad" : true
                        }
                    }
                }
            }
        }
    }
}
```

响应如下：

```
{
  "took" : 12,
  "timed_out" : false,
  "_shards" : {
    "total" : 10,
    "successful" : 10,
    "failed" : 0
  },
  "hits" : {
    "total" : 111331,
    "max_score" : 0.0,
    "hits" : [  ]
  },
  "aggregations" : {
    "my_date_histo" : {
      "buckets" : [ {
        "key_as_string" : "2015-12-24T02:00:00.000Z",
        "key" : 1450922400000,
        "doc_count" : 1462,
        "avgtime" : {
          "value" : 508.25649794801643
        }
      }, {
        ...
      }, {
        "key_as_string" : "2015-12-24T17:00:00.000Z",
        "key" : 1450976400000,
        "doc_count" : 1664,
        "avgtime" : {
          "value" : 504.7067307692308
        },
        "the_movavg" : {
          "value" : 500.9766851760192
        }
      }, {
        ...
      }, {
        "key_as_string" : "2015-12-25T09:00:00.000Z",
        "key" : 1451034000000,
        "doc_count" : 0,
        "the_movavg" : {
          "value" : 493.9519632950849,
          "value_as_string" : "1970-01-01T00:00:00.493Z"
        }
    } ]
  }
}
```

可以看到，在第一个移动窗口还没满足之前，是没有移动平均值的；而在实际数据已经结束以后，虽然没有平均值了，但是预测的移动平均值却还有数。

#### buckets_path 语法

由于 aggregation 是有堆叠层级关系的，所以 pipeline aggregation 在引用 metric aggregation 时也就会涉及到层级的问题。在上例中，`the_movavg` 和 `avgtime` 是同一层级，所以 `buckets_path` 直接写 `avgtime` 即可。但是如果我们把 `the_movavg` 上提一层，跟 `my_date_histo` 同级，这个 `buckets_path` 怎么写才行呢？

```
"buckets_path" : "my_date_histo>avgtime"
```

如果用的是返回的数值有多个值的聚合，比如 `percentiles` 或者 `extended_stats`，则是：

```
"buckets_path" : "percentile_over_time>percentile_one_time.95"
```

ES 目前能支持的聚合请求列表，参见：<https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations.html>。

#### See Also

Holt Winters 预测算法，见：<https://en.wikipedia.org/wiki/Holt-Winters>。其在运维领域最著名的运用是 RRDtool 中的 [HWPREDICT](http://rrdtool.org/rrdtool/doc/rrdtool.en.html)。

## search 请求参数

* from

从索引的第几条数据开始返回，默认是 0；

* size

返回多少条数据，默认是 10。

注意：Elasticsearch 集群实际是需要给 coordinate node 返回 `shards number * (from + size)` 条数据，然后在单机上进行排序，最后给客户端返回这个 size 大小的数据的。所以请谨慎使用 from 和 size 参数。

此外，Elasticsearch 2.x 还新增了一个索引级别的动态控制配置项：`index.max_result_window`，默认为 10000。即 `from + size` 大于 10000 的话，Elasticsearch 直接拒绝掉这次请求不进行具体搜索，以保护节点。

另外，Elasticsearch 2.x 还提供了一个小优化：当设置 `"size":0` 时，自动改变 `search_type` 为 count。跳过搜索过程的 fetch 阶段。

* timeout

coordinate node 等待超时时间。到达该阈值后，coordinate node 直接把当前收到的数据返回给客户端，不再继续等待 data node 后续的返回了。

** 注意：这个参数只是为了配合客户端程序，并不能取消掉 data node 上搜索任务还在继续运行和占用资源。**

* terminate\_after

各 data node 上，扫描单个分片时，找到多少条记录后，就认为足够了。这个参数可以切实保护 data node 上搜索任务不会长期运行和占用资源。但是也就意味着搜索范围没有覆盖全部索引，是一个抽样数据。准确率是不好判断的。

* request\_cache

各 data node 上，在分片级别，对请求的响应(仅限于 `hits.total` 数值、aggregation 和 suggestion 的结果集)做的缓存。注意：这个缓存的键值要求很严格，请求的 JSON 必须一字不易，缓存才能命中。

另外，`request_cache` 参数不能写在请求 JSON 里，只能以 URL 参数的形式存在。示例如下：

```
curl -XPOST http://localhost:9200/_search?request_cache=true -d '
{
    "size" : 0,
    "timeout" : "120s",
    "terminate_after" : 1000000,
    "query" : { "match_all" : {} },
    "aggs" : { "terms" : { "terms" : { "field" : "keyname" } } }
}
'
```

# script

Elasticsearch 中，可以使用自定义脚本扩展功能。包括评分、过滤函数和聚合字段等方面。内置脚本引擎历经 MVEL、Groovy、Lucene expression 的变换后，Elastic.co 最终决定实现一个自己专用的 Painless 脚本语言，并在 5.0 版正式发布。

作为 Elastic Stack 场景，我们只介绍在聚合字段方面使用 script 的方式。

## 动态提交

最简单易用的方式，就是在正常的请求体中，把 `field` 换成 `script` 提交。比如一个标准的 terms agg 改成 script 方式，写法如下：

```
# curl 127.0.0.1:9200/logstash-2015.06.29/_search -d '{
    "aggs" : {
        "clientip_top10" : {
            "terms" : {
                "script" : {
                    "lang" : "painless",
                    "inline" : "doc['clientip'].value"
                }
            }
        }
    }
}'
```

在 script 中，有三种方式引用数据：`doc['clientip'].value`、`_field['clientip'].value` 和 `_source.clientip`。其区别在于：

* `doc[].value` 读取 doc value 内的数据；
* `_field[]` 读取 field 设置 `"store":true` 的存储内容；
* `_source.obj.attr` 读取 `_source` 的 JSON 内容。

这也意味着，前者必须读取的是最终的词元字段数据，而后者可以返回任意的数据结构。

**注意**：如果有分词，且未禁用 fielddata 的话，`doc[].value` 读取到的是分词后的数据。所以请注意使用 `doc['clientip.keyword'].value` 写法。

## 固定文件

为了和动态提交的语法有区别，调用固定文件的写法如下：

```
# curl 127.0.0.1:9200/logstash-2015.06.29/_search -d '{
    "aggs" : {
        "clientip_subnet_top10" : {
            "terms" : {
                "script" : {
                   "file" : "getvalue",
                    "lang" : "groovy",
                    "params" : {
                        "fieldname": "clientip.keyword",
                        "pattern": "^((?:\d{1,3}\.?){3})\.\d{1,3}$"
                    }
                }
            }
        }
    }
}'
```

上例要求在 ES 集群的所有数据节点上，都保存有一个 `/etc/elasticsearch/scripts/getvalue.groovy` 文件，并且该脚本文件可以接收 `fieldname` 和 `pattern` 两个变量。试举例如下：

```
#!/usr/bin/env groovy
matcher = ( doc[fieldname].value =~ /${pattern}/ )
if (matcher.matches()) {
    matcher[0][1]
}
```

**注意**：ES 进程默认每分钟扫描一次 `/etc/elasticsearch/scripts/` 目录，并尝试加载该目录下所有文件作为 script。所以，不要在该目录内做文件编辑等工作，不要分发 .svn 等目录到生成环境，这些临时或者隐藏文件都会被 ES 进程加载然后报错。

## 其他语言

ES 支持通过插件方式，扩展脚本语言的支持，目前默认自带的语言包括：

* painless
* lucene expression
* groovy
* mustache

而 github 上目前已有以下语言插件支持，基本覆盖了所有 JVM 上的可用语言：

* <https://github.com/elastic/elasticsearch-lang-mvel>
* <https://github.com/elastic/elasticsearch-lang-javascript>
* <https://github.com/elastic/elasticsearch-lang-python>
* <https://github.com/hiredman/elasticsearch-lang-clojure>
* <https://github.com/felipehummel/elasticsearch-lang-scala>
* <https://github.com/fcheung/elasticsearch-jruby>


# reindex

Elasticsearch 本身不提供对索引的 rename，mapping 的 alter 等操作。所以，如果有需要对全索引数据进行导出，或者修改某个已有字段的 mapping 设置等情况下，我们只能通过 scroll API 导出全部数据，然后重新做一次索引写入。这个过程，叫做 reindex。

之前完成这个过程只能自己写程序或者用 logstash。5.0 中，Elasticsearch 将这个过程内置为 reindex API，但是要注意：这个接口并没有什么黑科技，其本质仅仅是将这段相同逻辑的代码预置分发而已。如果有复杂的数据变更操作等细节需求，依然需要自己编程完成。

下面分别给出这三种方法的示例：

## Perl 客户端

Elastic 官方提供各种语言的客户端库，其中，Perl 库提供了对 reindex 比较方便的写法和示例。通过 `cpanm Search::Elasticsearch` 命令安装库完毕后，使用以下程序即可：

```
use Search::Elasticsearch;
 
my $es   = Search::Elasticsearch->new(
    nodes => ['192.168.0.2:9200']
);
my $bulk = $es->bulk_helper(
    index   => 'new_index',
);
 
$bulk->reindex(
    source  => {
        index       => 'old_index',
        size        => 500,         # default
        search_type => 'scan'       # default
    }
);
```

## Logstash 做 reindex

在最新版的 Logstash 中，对 logstash-input-elasticsearch 插件做了一定的修改，使得通过 logstash 完成 reindex 成为可能。

reindex 操作的 logstash 配置如下：

```
input {
  elasticsearch {
    hosts => [ "192.168.0.2" ]
    index => "old_index"
    size => 500
    scroll => "5m"
    docinfo => true
  }
}
output {
  elasticsearch {
    hosts => [ "192.168.0.3" ]
    index => "%{[@metadata][_index]}"
    document_type => "%{[@metadata][_type]}"
    document_id => "%{[@metadata][_id]}"
  }
}
```

如果你做 reindex 的源索引并不是 logstash 记录的内容，也就是没有 `@timestamp`, `@version` 这两个 logstash 字段，那么可以在上面配置中添加一段 filter 配置，确保前后索引字段完全一致：

```
filter {
  mutate {
    remove_field => [ "@timestamp", "@version" ]
  }
}
```

## reindex API

简单的 reindex，可以很容易的完成：

```
curl -XPOST http://localhost:9200/_reindex -d '
{
  "source": {
    "index": "logstash-2016.10.29"
  },
  "dest": {
    "index": "logstash-new-2016.10.29"
  }
}'
```

复杂需求，也能通过配合其他 API，比如 script、pipeline 等来满足一些，下面举一个复杂的示例：

```
curl -XPOST http://localhost:9200/_reindex?requests_per_second=10000 -d '
{
  "source": {
    "remote": {
      "host": "http://192.168.0.2:9200",
    },
    "index": "metricbeat-*",
    "query": {
      "match": {
        "host": "webserver"
      }
    }
  },
  "dest": {
    "index": "metricbeat",
    "pipeline": "ingest-rule-1"
  },
  "script": {
    "lang": "painless",
    "inline": "ctx._index = 'metricbeat-' + (ctx._index.substring('metricbeat-'.length(), ctx._index.length())) + '-1'"
  }
}'
```

上面这个请求的作用，是将来自 192.168.0.2 集群的 metricbeat-2016.10.29 索引中，有关 `host:webserver` 的数据，读取出来以后，经过 localhost 集群的 `ingest-rule-1` 规则处理，在写入 localhost 集群的 metricbeat-2016.10.29-1 索引中。

注意：读取远端集群数据需要先配置对应的 `reindex.remote.whitelist:192.168.0.2:9200` 到 elasticsearch.yml 的白名单里。

通过 reindex 接口运行的任务可以通过同样是 5.0 新引入的任务管理接口进行取消、修改等操作。详细介绍见后续任务管理章节。

（本文完）

文本整理自《ELKstack权威指南》


