---
title:  Elasticsearch自动补齐建议-completion suggester
link_title: elasticsearch-completion-suggester
tags: [ElasticSearch]
date: 2016-08-08 14:37:49
categories: ElasticSearch
thumbnailImage: https://i.loli.net/2019/09/25/wOcjeg83JZbDn2E.png
thumbnailImagePosition: left
---
<!-- toc -->
<!-- more -->
![](https://i.loli.net/2019/09/25/wOcjeg83JZbDn2E.png)
## 1.mapping
    curl -XPUT 192.168.0.1:9200/person -d'  //新建一个persion的索引
    {
       "mappings": {
          "person": { //这个是_type
             "properties": {
                "name": {
                   "type": "string"
                }
                "tag_suggest": {
                   "type": "completion",  //设置为completion才能被suggest捕获
                   "index_analyzer": "ik",
                   "search_analyzer": "ik",
                   "payloads": false
                }
             }
          }
       }
    }'
	


## 2 .添加测试数据

    curl -XPUT  192.168.2.20:9200/person/person/1 -d'
    {
    "name": [
         "david",
        "jacky"
     ],
    "tag_suggest": {
        "input": [
         "david",
         "jacky"
      ]
      }
    }'
    curl -XPUT 192.168.0.1:9200/person/person/1 -d'
    {
    "name": [
        "andy",
        "jackson"
    ],
     "tag_suggest": {
    "input": [
         "andy",
        "jackson"
      ]
    }
    }'

## 3.DSL
    curl -XPOST 192.168.0.1:9200/person/_suggest -d'
    {
     "person_suggest":{
          "text":"jack",
         "completion": {
              "field" : "tag_suggest"
        }
    }
    }'

## 4.结果
    {
     "_shards": {
      "total": 1,
      "successful": 1,
      "failed": 0
     },
     "person_suggest": [
      {
         "text": "word",
         "offset": 0,
         "length": 4,
         "options": [
            {
               "text": "jacky",
               "score": 1
            },
            {
               "text": "jackson",
               "score": 1
            }
         ]
      }
    ]
    }
## 5.代码
```java
        CompletionSuggestionBuilder completionSuggestionBuilder = new         
        CompletionSuggestionBuilder("complete");
        completionSuggestionBuilder.text(paramMap.get("text"));
        completionSuggestionBuilder.field(paramMap.get("field"));
        completionSuggestionBuilder.size(10);

	    IElasticsearchClient client = index.getIndexClient();
    	CompletionSuggestionBuilder completionSuggestion = completionSuggestionBuilder 
		SuggestResponse resp = client.prepareSuggest(realIndexName)
				.addSuggestion(completionSuggestion).execute().actionGet();

 	    List<? extends Suggest.Suggestion.Entry<? extends Suggest.Suggestion.Entry.Option>>            list =         response.getSuggest().getSuggestion("complete").getEntries();
        List<String> suggestList = new ArrayList<String>();
        if (list == null) {
            return null;
        } else {
            for (Suggest.Suggestion.Entry<? extends Suggest.Suggestion.Entry.Option> e : list){
                for (Suggest.Suggestion.Entry.Option option : e) {
                    suggestList.add(option.getText().toString());
                }
            }
```

