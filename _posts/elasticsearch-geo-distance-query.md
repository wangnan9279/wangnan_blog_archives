---
title: Elasticsearch地理位置查询-Geo Distance Range Query
tags: [ElasticSearch, 搜索引擎]
date: 2017-03-17 13:50:17
categories: ElasticSearch
link_title: elasticsearch-geo-distance-query
toc: true
---
![](http://onxkn9cbz.bkt.clouddn.com/elasticsearch.png)
> ## 官方文档：
https://www.elastic.co/guide/en/elasticsearch/reference/2.4/query-dsl-geo-distance-query.html#_geohash_3



## 索引mapping定义:
索引中定义一个字段pin，添加一个属性location，type为geo_point

	"pin" : {
	  "properties" : {
	    "location" : {
	      "type" : "geo_point"
		    }
		  }
		}

<!-- more -->
## DSL:
报文中的包含一个match all的query  , filter中的distance指定了距离范围，pin.location是经纬度

	{
    "bool" : {
        "must" : {
            "match_all" : {}
        },
        "filter" : {
            "geo_distance" : {
                "distance" : "200km",
                "pin.location" : {
                    "lat" : 40,
                    "lon" : -70
                }
            }
        }
    }
	}



## 代码：
### 拼装 query 和 sort
```java
		   QueryBuilder builder = new GeoDistanceRangeQueryBuilder （"pin.location"） 
	                .point(lat,lon)
	                .from("0km")  
	                .to("10000km")  
	                .includeLower(true)  
	                .includeUpper(false)  
	                .optimizeBbox("memory")  
	                .geoDistance(GeoDistance.ARC);  
	 
	          GeoDistanceSortBuilder sort = SortBuilders.geoDistanceSort("location");  
	          GeoDistanceSortBuilder sort = new GeoDistanceSortBuilder("location");  
	          sort.unit(DistanceUnit.KILOMETERS); 
	          sort.order(SortOrder.ASC);  
	          sort.point(lat,lon); 
```

### 构造dsl生产器
```java
	
			SearchSourceBuilder sourceBuilder = SearchSourceBuilder.searchSource(); 
			sourceBuilder.query(QueryBuilders.boolQuery())			      
			.must(builder)
            sourceBuilder.sort(sort);
```

### 调用elasticsearch
```java
	final SearchResponse response = executeGet(new ClientCallback<SearchResponse>() {
            @Override
            public ActionFuture<SearchResponse> execute(final Client client) {
		    final SearchSourceBuilder sourceBuilder =    SearchParamUtils
		    .genSearchSourceBuilderFromSearchParam(searchParam);  
            String[] indexNames = new String[aliasIndexNameList.size()];
	        SearchRequest request = Requests.searchRequest(aliasIndexNameList.toArray(indexNames))
	        .types(type);
            request.source(sourceBuilder.toString());
            if (searchParam.getSearchType() != null) {
                    request.searchType(searchParam.getSearchType());
            }
                return client.search(request);
            }
        });
```
	      
	      


### 获取每条记录的距离
```java
			SearchResult searchResult = new SearchResult();
	        SearchHits hits = response.getHits();
	        for (SearchHit hit : hits.getHits()) {
	        Object[] sortArray = hit.getSortValues();
                if(sortArray!=null&&sortArray.length>0){
                    BigDecimal geoDis = new BigDecimal((Double) sortArray[sortArray.length-1]);
                    map.put("geoDistance", geoDis.setScale(0, BigDecimal.ROUND_HALF_DOWN));
                    		System.out.println("距离" + hit.getSource().get("geoDistance"));
                }
			}
			
```
