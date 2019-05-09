---
title: ES中的分析器和IK分词器插件
link_title: elasticsearch-analyzer
date: 2017-09-18 15:50:59
tags: [ElasticSearch]
categories: ElasticSearch
thumbnailImage: https://upload-images.jianshu.io/upload_images/79431-27660e8a42b51714.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/513/format/webp	
thumbnailImagePosition: left
---
<span/>
<!-- more -->
![](https://upload-images.jianshu.io/upload_images/79431-27660e8a42b51714.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/513/format/webp)
<!-- toc -->

# 一些概念
## Token（词元）
全文搜索引擎会用某种算法对要建索引的文档进行分析， 从文档中提取出若干Tokenizer(分词器)
## Tokenizer(分词器)
这些算法叫做Tokenizer(分词器)
## Token Filter(词元处理器)
这些Token会被进一步处理， 比如转成小写等， 这些处理算法被称为TokenFilter(词元处理器)
## Term(词)
被处理后的结果被称为Term(词)
## Character Filter(字符过滤器)
文本被Tokenizer处理前可能要做一些预处理， 比如去掉里面的HTML标记， 这些处理的算法被称为Character Filter(字符过滤器)
##  Analyzer(分析器)
这整个的分析算法被称为Analyzer(分析器)

Analyzer（分析器）由Tokenizer（分词器）和Filter（过滤器）组成

## 图片
![image](https://upload-images.jianshu.io/upload_images/79431-d65616497a39d77b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/530/format/webp)

# ES中的分词器
## ES内置分析器
- standard analyzer
- simple analyzer
- stop analyzer
- keyword analyzer
- pattern analyzer
- language analyzers
- snowball analyzer
- custom analyzer

## ES内置分析器
- standard tokenizer
- edge ngram tokenizer
- keyword tokenizer
- letter analyzer
- lowercase analyzer
- ngram analyzers
- whitespace analyzer
- pattern analyzer
- uax email url analyzer
- path hierarchy analyzer	

## ES内置过滤器
- standard filter	
- ascii folding filter	
- length filter	
- lowercase filter	
- ngram filter	
- edge ngram filter
- porter stem filter	
- shingle filter
- stop filter	stop	
- word delimiter filter	
- stemmer token filter	
- stemmer override filter	
- keyword marker filter	
- keyword repeat filter
- kstem filter		
- snowball filter		
- phonetic filter	
- synonym filter		
- compound word filter	 
- reverse filter	
- elision filter
- truncate filter	
- unique filter	
- pattern capture  filter		
- pattern replace filter	
- trim filter
- limit token count filter	
- hunspell filter	
- common grams filter	
- normalization filter	

## ES内置的character filter
- mapping char filter  根据配置的映射关系替换字符
- html strip char filter 去掉HTML元素
- pattern replace char filter  用正则表达式处理字符串

# 自定义分析器
ES允许用户通过配置文件elasticsearch.yml自定义分析器Analyzer

```xml
    index:
           analysis:
                     analyzer:
                            myAnalyzer:
                                   tokenizer: standard
                                   filter: [standard, lowercase, stop]
```

也可以使用第三方分析器，比如IKAnalyzer

# IKAnalyzer
## IK简介
IK Analyzer是一个开源的，基于java语言开发的轻量级的中文分词工具包。从2006年12月推出1.0版开始， IKAnalyzer已经推出了4个大版本。最初，它是以开源项目Luence为应用主体的，结合词典分词和文法分析算法的中文分词组件。从3.0版本开 始，IK发展为面向Java的公用分词组件，独立于Lucene项目，同时提供了对Lucene的默认优化实现。在2012版本中，IK实现了简单的分词 歧义排除算法，标志着IK分词器从单纯的词典分词向模拟语义分词衍化。  

## IK Analyzer 2012特性
1.采用了特有的“正向迭代最细粒度切分算法“，支持细粒度和智能分词两种切分模式；

2.在系统环境：Core2 i7 3.4G双核，4G内存，window 7 64位， Sun JDK 1.6_29 64位 普通pc环境测试，IK2012具有160万字/秒（3000KB/S）的高速处理能力。

3.2012版本的智能分词模式支持简单的分词排歧义处理和数量词合并输出。

4.采用了多子处理器分析模式，支持：英文字母、数字、中文词汇等分词处理，兼容韩文、日文字符

5.优化的词典存储，更小的内存占用。支持用户词典扩展定义。特别的，在2012版本，词典支持中文，英文，数字混合词语。

## 安装
1. 通过git clone https://github.com/medcl/elasticsearch-analysis-ik，下载分词器源码
2. 执行命令：mvn clean package，打包生成elasticsearch-analysis-ik-1.2.5.jar
3. 将这个jar拷贝到ES_HOME/plugins/analysis-ik目录下面，如果没有该目录，则先创建该目录
4. ES_HOME/config/elasticsearch.yml文件在文件最后加入如下内容：
```xml
index:
  analysis:                   
    analyzer:      
      ik:
          alias: [ik_analyzer]
          type: org.elasticsearch.index.analysis.IkAnalyzerProvider
      ik_max_word:
          type: ik
          use_smart: false
      ik_smart:
          type: ik
          use_smart: true
index.analysis.analyzer.default.type: ik
```

## 测试
1. 创建一个索引，名为index。
```xml
curl -XPUT http://localhost:9200/index
```
2. 为索引index创建mapping
```xml
curl -XPOST http://localhost:9200/index/fulltext/_mapping -d'
{
    "fulltext": {
             "_all": {
            "analyzer": "ik"
        },
        "properties": {
            "content": {
                "type" : "string",
                "boost" : 8.0,
                "term_vector" : "with_positions_offsets",
                "analyzer" : "ik",
                "include_in_all" : true
            }
        }
    }
}'
```
3. 测试
```xml
curl 'http://localhost:9200/index/_analyze?analyzer=ik&pretty=true' -d '
{
"text":"世界如此之大"
}'
```
4.显示结果
```xml
{
  "tokens" : [ {
    "token" : "text",
    "start_offset" : 4,
    "end_offset" : 8,
    "type" : "ENGLISH",
    "position" : 1
  }, {
    "token" : "世界",
    "start_offset" : 11,
    "end_offset" : 13,
    "type" : "CN_WORD",
    "position" : 2
  }, {
    "token" : "如此",
    "start_offset" : 13,
    "end_offset" : 15,
    "type" : "CN_WORD",
    "position" : 3
  }, {
    "token" : "之大",
    "start_offset" : 15,
    "end_offset" : 17,
    "type" : "CN_WORD",
    "position" : 4
  } ]
}
```
  