---
title: Redis数据安全与性能保障
tags: [Redis]
date: 2016-08-23 16:14:12
categories: Redis
link_title: redis-data-safety
thumbnailImage: https://i.loli.net/2019/09/25/NLX2v7dwDOUfrnT.png
thumbnailImagePosition: left
---
<!-- toc -->
<!-- more -->
![](https://i.loli.net/2019/09/25/NLX2v7dwDOUfrnT.png)

<!-- more -->
## 持久化选项
![01](redis-data-safety/01.png)

## 复制
![02](redis-data-safety/02.png)
## 处理系统故障
![03](redis-data-safety/03.png)
## redis事务
![04](redis-data-safety/04.png)
## 非事务型流水线
可以接受多个参数的添加命令和更新命令，比如：MGET,MSET,HMGET,HMSET,RPUSH,LPUSH,SADD,ZADD,这些命令简化了那些需要重复执行相同命令的操作，而极大的提升了性能
## 性能方面注意事项
可以用性能测试程序redis-benchmark来测试

（注：内容整理自《redis实战》）