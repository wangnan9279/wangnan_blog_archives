---
title: Logstash中的Grok正则捕获
link_title: logstash-grok
date: 2018-09-08 09:49:48
tags: [Logstash]
categories: ELKstack
thumbnailImage: https://i.loli.net/2019/09/24/y8pX6aMDFxSP9OC.png
thumbnailImagePosition: left
---
<span/>
<!-- more -->
![](https://i.loli.net/2019/09/24/y8pX6aMDFxSP9OC.png)
<!-- toc -->
# 概述
Grok 是 Logstash 最重要的插件。你可以在 grok 里预定义好命名正则表达式

Grok 支持把预定义的 grok 表达式 写入到文件中，官方提供的预定义 grok 表达式见：https://github.com/logstash-plugins/logstash-patterns-core/tree/master/patterns。

grok的语法格式为 %{SYNTAX:SEMANTIC}

SYNTAX是文本要匹配的模式

SEMANTIC 是匹配到的文本片段的标识


例如：

```
%{NUMBER:duration}
%{IP:client}
```


默认情况下，所有的SEMANTIC是以字符串的方式保存，如果想要转换一个SEMANTIC的数据类型，例如转换一个字符串为整形，可以写成如下的方式：

%{NUMBER:num:int}

例如日志

```
55.3.244.1 GET /index.html 15824 0.043
```


可以写成如下的grok过滤表达式

```
%{IP:client} %{WORD:method} %{URIPATHPARAM:request} %{NUMBER:bytes} %{NUMBER:duration}
```




#  示例

## %{COMBINEDAPACHELOG}

%{COMBINEDAPACHELOG} 是logstash自带的匹配模式

它的grok表达式是：

```
COMMONAPACHELOG %{IPORHOST:clientip} %{USER:ident} %{USER:auth} \[%{HTTPDATE:timestamp}\] "(?:%{WORD:verb} %{NOTSPACE:req
uest}(?: HTTP/%{NUMBER:httpversion})?|%{DATA:rawrequest})" %{NUMBER:response} (?:%{NUMBER:bytes}|-)
COMBINEDAPACHELOG %{COMMONAPACHELOG} %{QS:referrer} %{QS:agent}
```

输入常规的Apache日志：

```
127.0.0.1 - - [13/Apr/2015:17:22:03 +0800] "GET /router.php HTTP/1.1" 404 285 "-" "curl/7.19.7 (x86_64-redhat-linux-gnu) libcurl/7.19.7 NSS/3.15.3 zlib/1.2.3 libidn/1.18 libssh2/1.4.2"
127.0.0.1 - - [13/Apr/2015:17:22:03 +0800] "GET /router.php HTTP/1.1" 404 285 "-" "curl/7.19.7 (x86_64-redhat-linux-gnu) libcurl/7.19.7 NSS/3.15.3 zlib/1.2.3 libidn/1.18 libssh2/1.4.2"
```
配置filter：

```
filter {
  if [type] == "apache" {
     grok {
          match => ["message",  "%{COMBINEDAPACHELOG}"]
          }
                         }
       }
```

输出：

```
{
        "message" => "127.0.0.1 - - [14/Apr/2015:09:53:40 +0800] \"GET /router.php HTTP/1.1\" 404 285 \"-\" \"curl/7.19.7 (x86_64-redhat-linux-gnu) libcurl/7.19.7 NSS/3.15.3 zlib/1.2.3 libidn/1.18 libssh2/1.4.2\"",
       "@version" => "1",
     "@timestamp" => "2015-04-14T01:53:57.182Z",
           "type" => "apache",
           "host" => "xxxxxxxx",
           "path" => "/var/log/httpd/access_log",
       "clientip" => "127.0.0.1",
          "ident" => "-",
           "auth" => "-",
      "timestamp" => "14/Apr/2015:09:53:40 +0800",
           "verb" => "GET",
        "request" => "/router.php",
    "httpversion" => "1.1",
       "response" => "404",
          "bytes" => "285",
       "referrer" => "\"-\"",
          "agent" => "\"curl/7.19.7 (x86_64-redhat-linux-gnu) libcurl/7.19.7 NSS/3.15.3 zlib/1.2.3 libidn/1.18 libssh2/1.4.2\""
}
{
        "message" => "127.0.0.1 - - [14/Apr/2015:09:53:40 +0800] \"GET /router.php HTTP/1.1\" 404 285 \"-\" \"curl/7.19.7 (x86_64-redhat-linux-gnu) libcurl/7.19.7 NSS/3.15.3 zlib/1.2.3 libidn/1.18 libssh2/1.4.2\"",
       "@version" => "1",
     "@timestamp" => "2015-04-14T01:53:57.187Z",
           "type" => "apache",
           "host" => "xxxxxxx",
           "path" => "/var/log/httpd/access_log",
       "clientip" => "127.0.0.1",
          "ident" => "-",
           "auth" => "-",
      "timestamp" => "14/Apr/2015:09:53:40 +0800",
           "verb" => "GET",
        "request" => "/router.php",
    "httpversion" => "1.1",
       "response" => "404",
          "bytes" => "285",
       "referrer" => "\"-\"",
          "agent" => "\"curl/7.19.7 (x86_64-redhat-linux-gnu) libcurl/7.19.7 NSS/3.15.3 zlib/1.2.3 libidn/1.18 libssh2/1.4.2\""
}
```


## 官方文档示例

下面是从官方文件中摘抄的最简单但是足够说明用法的示例：

```
USERNAME [a-zA-Z0-9._-]+
USER %{USERNAME}
```
第一行，用普通的正则表达式来定义一个 grok 表达式
第二行，通过打印赋值格式(sprintf format)，用前面定义好的 grok 表达式来定义另一个 grok 表达式

grok 表达式的打印赋值格式的完整语法是下面这样的：

```
%{PATTERN_NAME:capture_name:data_type}
```

我们的配置filter成下面这样：

```
filter {
    grok {
        match => {
            "message" => "%{WORD} %{NUMBER:request_time:float}%{WORD}"
        }
    }
}
```
运行 logstash 进程然后输入 "begin 123.456 end"

会看到类似下面这样的输出：

```
{
         "message" => "begin 123.456 end",
        "@version" => "1",
      "@timestamp" => "2014-08-09T12:23:36.634Z",
            "host" => "raochenlindeMacBook-Air.local",
    "request_time" => 123.456
}
```

实际运用中，我们需要处理各种各样的日志文件，如果你都是在配置文件里各自写一行自己的表达式，就完全不可管理了。所以，我们建议是把所有的 grok 表达式统一写入到一个地方。然后用 filter/grok 的 patterns_dir 选项来指明。

如果你把 "message" 里所有的信息都 grok 到不同的字段了，数据实质上就相当于是重复存储了两份。所以你可以用 remove_field 参数来删除掉 message 字段，或者用 overwrite 参数来重写默认的 message 字段，只保留最重要的部分。


```
filter {
    grok {
        patterns_dir => ["/path/to/your/own/patterns"]
        match => {
            "message" => "%{SYSLOGBASE} %{DATA:message}"
        }
        overwrite => ["message"]
    }
}
```

建议每个人都要使用 Grok Debugger 来调试自己的 grok 表达式。
https://grokdebug.herokuapp.com/

## 自定义匹配
在有些情况下自带的匹配模式无法满足需求，可以自定义一些匹配模式

首先可以根据正则表达式匹配文本片段

```
(?<field_name>the pattern here)
```
例如，postfix日志有一个字段表示 queue id，可以使用以下表达式进行匹配：


```
(?<queue_id>[0-9A-F]{10,11}
```

可以手动创建一个匹配文件，内容：

```
# contents of ./patterns/postfix:
POSTFIX_QUEUEID [0-9A-F]{10,11}
```
filter配置：

```
 filter {
       grok {
         patterns_dir => "./patterns"
         match => [ "message", "%{SYSLOGBASE} %{POSTFIX_QUEUEID:queue_id}: %{GREEDYDATA:syslog_message}" ]
       }
     }
```
patterns_dir指定了文件的目录，match中使用了自定义的：POSTFIX_QUEUEID

输入：

```
55.3.244.1 GET /index.html 15824 0.043 ABC24C98567
```

输出：

```
client_id_address: 55.3.244.1
method: GET
request: /index.html
bytes: 15824
http_response_time: 0.043
queue_id: ABC24C98567
```
发现queue_id 被匹配出来了

# 常用内置方法
## add_field
当pattern匹配切分成功之后，可以动态的对某些字段进行特定的修改或者添加新的字段，使用%{fieldName}来获取字段的值

filter：

```
filter {
grok{
add_field => { "foo_%{somefield}" => "Hello world, %{somefield}" }
}
}

```
如果somefield=dad，logstash会将foo_dad新字段加入elasticsearch，并将值Hello world, dad赋予该字段


## add_tag
为经过filter或者匹配成功的event添加标签

```
filter {
grok {
add_tag => [ "foo_%{somefield}" ]
}
}
```
