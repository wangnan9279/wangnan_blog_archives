---
title: Elasticsearch使用过滤器优化查询
tags: [ElasticSearch, 搜索引擎, 读书笔记]
date: 2017-03-23 16:12:14
categories: ElasticSearch
link_title: elasticsearc-filtered
---
> elasticsearch提供了一种特殊的缓存，即过滤器缓存（filter cache），用来储存过滤器的结果

<!-- more -->

被缓存的过滤器不需要消耗过多的内存，因为他们只储存了哪些文档能与过滤器相匹配的相关信息，而且可供后续所有与之相关的查询重复使用，从而极大的提高了查询性能

执行下面这个查询：
```json
{
    "query":{
        "bool":{
            "must":[
            {
                "term":{"name":"joe"}    
            },
            {
                "term":{"year":1981}
            }
            ]
        }
    }
}
```
该查询能查询出满足指定姓名和出生年代条件的足球运动员，只有同时满足两个条件的查询才可以被缓存起来。

优化这个查询：
人名有太多可能性，它不是完美的缓存候选对象，而年代是，我们使用另一种查询方法，该查询组合了查询类型与过滤器：

```json
{
    "query":{
        "filtered":{
            "query":{
                "term"：{"name":"joe"}
            },
            "filter":{
                "term":{"year":1981}
            }
        }
    }
}
```
第一次执行该查询以后，过滤器会被es缓存起来，如果后续的其他查询也要使用该过滤器，则她会被重复使用，避免es重复加载相关数据