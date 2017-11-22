---
title: logstash用kafka作为输入源
link_title: logstash-kafka-input
date: 2017-11-16 16:06:41
tags: [Logstash]
categories: ELKstack
thumbnailImage: http://onxkn9cbz.bkt.clouddn.com/04.jpg	
thumbnailImagePosition: left
---
<span/>
<!-- more -->
![](http://onxkn9cbz.bkt.clouddn.com/04.jpg)
<!-- toc -->
# 安装

依赖:jdk7及以上版本 

Logstash版本:2.3.4

步骤：
1. 官网下载tar.gz包

链接地址：[链接](https://www.elastic.co/cn/downloads/logstash)
2. 扔到机器上解压
```
tar zxvf logstash-2.3.4.tar.gz
```
3.安装kafka input插件，顺便把output也安装了

```
bin/logstash-plugin install logstash-output-kafka
bin/logstash-plugin install logstash-input-kafka
```

# 配置 

bin文件夹中添加文件 logstash.conf

```
input {
    kafka {
        zk_connect => "xxx:2181"
        topic_id => "xxx"
        reset_beginning => true
        consumer_threads => 5  
        decorate_events => false 
        }
    }
	
output {
        elasticsearch {
			action => "index"
	        hosts => ["xxx"]
            index => "logs-%{+YYYY.MM.dd}"
			document_type => "logs"
		}
}
```

## input中参数解释
- zk_connect

kafka连接的zk地址，通过这个配置项获取kafka的数据


- group_id

消费者分组，可以通过组 ID 去指定，不同的组之间消费是相互不受影响的，相互隔离。

- topic_id

指定消费话题，也是必填项目，指定消费某个 topic ，这个其实就是订阅某个主题，然后去消费。

- reset_beginning

logstash 启动后从什么位置开始读取数据，默认是结束位置，也就是说 logstash 进程会以从上次读取结束时的偏移量开始继续读取，如果之前没有消费过，那么就开始从头读取.如果你是要导入原有数据，把这个设定改成 "true"， logstash 进程就从头开始读取.有点类似 cat ，但是读到最后一行不会终止，而是变成 tail -F ，继续监听相应数据。

- decorate_events

在输出消息的时候会输出自身的信息包括:消费消息的大小， topic 来源以及 consumer 的 group 信息。

- rebalance_max_retries

当有新的 consumer(logstash) 加入到同一 group 时，将会 reblance ，此后将会有 partitions 的消费端迁移到新的 consumer 上，如果一个 consumer 获得了某个 partition 的消费权限，那么它将会向 zookeeper 注册， Partition Owner registry 节点信息，但是有可能此时旧的 consumer 尚没有释放此节点，此值用于控制，注册节点的重试次数。

- consumer_timeout_ms

指定时间内没有消息到达就抛出异常，一般不需要改。

以上是相对重要参数的使用示例，更多参数可以选项可以跟据 https://github.com/joekiller/logstash-kafka/blob/master/README.md 查看 input 默认参数。


# 启动

```
bin/logstash -f logstash.conf &
```

启动后es中将会写入指定topic中的日志数据

还可以自定义es的模板和配置logstash的filter截取一些需要的业务字段，此处不详细说明


