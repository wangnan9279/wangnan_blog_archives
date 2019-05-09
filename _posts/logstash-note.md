---
title: Logstash学习笔记
link_title: logstash-note
date: 2018-09-01 17:06:12
tags: [Logstash]
categories: ELKstack
thumbnailImage: https://upload-images.jianshu.io/upload_images/79431-242d4984f703f03f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/358/format/webp	
thumbnailImagePosition: left
---
<span/>
<!-- more -->
![](https://upload-images.jianshu.io/upload_images/79431-242d4984f703f03f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/358/format/webp)
<!-- toc -->

# 背景
先介绍下ELK stack

**Elasticsearch** 

Elasticsearch 是基于 JSON 的分布式搜索和分析引擎，专为实现水平扩展、高可用和管理便捷性而设计

**Logstash**

Logstash 是动态数据收集管道，拥有可扩展的插件生态系统，能够与 Elasticsearch 产生强大的协同作用。

**Kibana**

Kibana 能够以图表的形式呈现数据，并且具有可扩展的用户界面，供您全方位配置和管理 Elastic Stack

**ELK典型使用场景**

用Elasticsearch作为后台数据的存储，kibana用来前端的报表展示。Logstash在其过程中担任搬运工的角色


我们先把logstash安装然后简单用起来，然后详细解析logstash

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
安装完成

## 简单使用示例
### 1.命令行输入内容然后让logstash输出内容

进到bin目录下运行：

```
./logstash -e 'input { stdin { } } output { stdout {} }'
```
logstach输出：

```
Settings: Default pipeline workers: 2
Pipeline main started
```
表示启动成功

在控制台输入123，然后logstash输出如下：

```
123
2017-11-02T02:42:50.836Z localhost.localdomain 123
```

解释：

每位系统管理员都肯定写过很多类似这样的命令：

```
cat randdata | awk '{print $2}' | sort | uniq -c | tee sortdata。
```


这个管道符 | 可以算是 Linux 世界最伟大的发明之一(另一个是“一切皆文件”)。
Logstash 就像管道符一样！

你输入(就像命令行的 cat )数据，然后处理过滤(就像 awk 或者 uniq 之类)数据，最后输出(就像 tee )到其他地方。

当然实际上，Logstash 是用不同的线程来实现这些的

数据在线程之间以 事件 的形式流传。不要叫行，因为 logstash 可以处理多行事件。

Logstash 会给事件添加一些额外信息。最重要的就是 @timestamp，用来标记事件的发生时间。因为这个字段涉及到 Logstash 的内部流转，所以必须是一个 joda 对象，如果你尝试自己给一个字符串字段重命名为 @timestamp 的话，Logstash 会直接报错。所以，请使用 filters/date 插件 来管理这个特殊字段

此外，大多数时候，还可以见到另外几个：
- host 标记事件发生在哪里。
- type 标记事件的唯一类型。
- tags 标记事件的某方面属性。这是一个数组，一个事件可以有多个标签。


-e参数允许Logstash直接通过命令行接受设置。这点尤其快速的帮助我们反复的测试配置是否正确而不用写配置文件

以上例子我们在运行logstash中，定义了一个叫"stdin"的input还有一个"stdout"的output，无论我们输入什么字符，Logstash都会按照某种格式来返回我们输入的字符




### 2.命令行输入内容然后让logstash以某种格式输出

运行命令：

```
./logstash -e 'input { stdin { } } output { stdout { codec => rubydebug } }'  
```

logstach输出：

```
Settings: Default pipeline workers: 2
Pipeline main started
```
表示启动成功

在控制台输入123，然后logstash输出如下：

```
123
{
       "message" => "123",
      "@version" => "1",
    "@timestamp" => "2017-11-02T02:51:55.681Z",
          "host" => "localhost.localdomain"
}

```
命令解释：
codec 指定了数据输出类型是rubydebug类型，还可以是json类型等等

### 3.命令行输入内容然后让logstash使用Elasticsearch存储

-e参数在命令行中指定配置是很常用的方式，不过如果需要配置更多设置则需要很长的内容。这种情况，我们首先创建一个简单的配置文件

建一个文件名是"logstash.conf"的配置文件并且保存在和Logstash相同的目录中,配置一个已经按照好的es的ip


```
input { stdin { } }  
output {  
  elasticsearch { host => "xxx.xxx.xxx.xx" }  
  stdout { codec => rubydebug }  
} 
```

到bin目录运行logstash -f参数用于指定配置文件：

```
.logstash -f logstash.conf
```
logstach输出：

```
Settings: Default pipeline workers: 2
Pipeline main started
```
表示启动成功

输入123，logstach输出如下

```
123
{
       "message" => "123",
      "@version" => "1",
    "@timestamp" => "2017-11-02T03:02:48.516Z",
          "host" => "localhost.localdomain"
}

```
然后到es的head插件中查看,es中已经存在一条数据了：
![image](https://upload-images.jianshu.io/upload_images/79431-61523e15c096eccb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/487/format/webp)

你将发现Logstash可以足够灵巧的在Elasticsearch上建立索引... 每天会按照默认格式是logstash-YYYY.MM.DD来建立索引。在午夜(GMT)，Logstash自动按照时间戳更新索引。我们可以根据追溯多长时间的数据作为依据来制定保持多少数据，当然你也可以把比较老的数据迁移到其他的地方(重新索引)来方便查询，


### 4.使用过滤器解析日志存到es中

在bin目录下创建一个配置文件 logstash-filter.conf

```
input { stdin { } }
  
filter {  
  grok {  
    match => { "message" => "%{COMBINEDAPACHELOG}" }  
  }  
  date {  
    match => [ "timestamp" , "dd/MMM/yyyy:HH:mm:ss Z" ]  
  }  
}  
  
output {  
  elasticsearch { hosts => "192.168.102.13" }
  stdout { codec => rubydebug }  
} 

```
运行：

```
./logstash -f logstash-filter.conf
```
logstach输出：

```
Settings: Default pipeline workers: 2
Pipeline main started
```
表示启动成功

输入：

```
127.0.0.1 - - [11/Dec/2013:00:01:45 -0800] "GET /xampp/status.php HTTP/1.1" 200 3891 "http://cadenza/xampp/navi.php" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.9; rv:25.0) Gecko/20100101 Firefox/25.0"  
```

logstash输出：

```
{
        "message" => "127.0.0.1 - - [11/Dec/2013:00:01:45 -0800] \"GET /xampp/status.php HTTP/1.1\" 200 3891 \"http://cadenza/xampp/navi.php\" \"Mozilla/5.0 (Macintosh; Intel Mac OS X 10.9; rv:25.0) Gecko/20100101 Firefox/25.0\"  ",
       "@version" => "1",
     "@timestamp" => "2013-12-11T08:01:45.000Z",
           "host" => "localhost.localdomain",
       "clientip" => "127.0.0.1",
          "ident" => "-",
           "auth" => "-",
      "timestamp" => "11/Dec/2013:00:01:45 -0800",
           "verb" => "GET",
        "request" => "/xampp/status.php",
    "httpversion" => "1.1",
       "response" => "200",
          "bytes" => "3891",
       "referrer" => "\"http://cadenza/xampp/navi.php\"",
          "agent" => "\"Mozilla/5.0 (Macintosh; Intel Mac OS X 10.9; rv:25.0) Gecko/20100101 Firefox/25.0\""
}
```
然后到es的head插件中查看,es中已经存在一条数据了：
![image](https://upload-images.jianshu.io/upload_images/79431-cb0cb0ec9b07fbc9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/603/format/webp)

Logstash(使用了grok过滤器)能够将一行的日志数据(Apache的"combined log"格式)分割设置为不同的数据字段。这一点对于日后解析和查询我们自己的日志数据非常有用。比如：HTTP的返回状态码，IP地址相关等等，非常的容易。很少有匹配规则没有被grok包含，所以如果你正尝试的解析一些常见的日志格式，或许已经有人为了做了这样的工作。

另外一个过滤器是date filter。这个过滤器来负责解析出来日志中的时间戳并将值赋给timestamp字段


### 5.从文件获取日志保存到es中
在bin目录下创建一个配置文件 logstash-apache.conf

```
input {
  file {  
    path => "/opt/logstash-2.3.4/logs/*"          
    start_position => beginning  
  }  
}  
  
filter {                                                 
    grok {  
      match => { "message" => "%{COMBINEDAPACHELOG}" }  
    }  
  date {  
    match => [ "timestamp" , "dd/MMM/yyyy:HH:mm:ss Z" ]  
  }  
}  
  
output {  
  elasticsearch {  
    hosts => "192.168.102.13"
  }  
  stdout { codec => rubydebug }  
}  

```
运行：

```
./logstash -f logstash-apache.conf
```
logstach输出：

```
Settings: Default pipeline workers: 2
Pipeline main started
```
表示启动成功

去es插件中可以看到apache的日志数据已经导入到ES中了

此外，你也许还会发现Logstash不会重复处理文件中已经处理过得events。因为Logstash已经记录了文件处理的位置，这样就只处理文件中新加入的行数


# logstash详解
## 引言
从 Logstash 的名字就能看出，它主要负责跟日志相关的各类操作，在此之前，我们先来看看日志管理的三个境界吧

- 境界一
『昨夜西风凋碧树。独上高楼，望尽天涯路』，在各台服务器上用传统的 linux 工具（如 cat, tail, sed, awk, grep 等）对日志进行简单的分析和处理，基本上可以认为是命令级别的操作，成本很低，速度很快，但难以复用，也只能完成基本的操作。

- 境界二
『衣带渐宽终不悔，为伊消得人憔悴』，服务器多了之后，分散管理的成本变得越来越多，所以会利用 rsyslog 这样的工具，把各台机器上的日志汇总到某一台指定的服务器上，进行集中化管理。这样带来的问题是日志量剧增，小作坊式的管理基本难以满足需求。

- 境界三
『众里寻他千百度，蓦然回首，那人却在灯火阑珊处』，随着日志量的增大，我们从日志中获取去所需信息，并找到各类关联事件的难度会逐渐加大，这个时候，就是 Logstash 登场的时候了

Logstash 的主要优势，一个是在支持各类插件的前提下提供统一的管道进行日志处理（就是 input-filter-output 这一套），二个是灵活且性能不错

logstash里面最基本的概念（先不管codec）

logstash收集日志基本流程: input-->filter-->output 

input:从哪里收集日志

filter:对日志进行过滤

output:输出哪里

## 架构
Logstash 是由 JRuby 编写的，使用基于消息的简单架构，在 JVM 上运行。理念非常简单，如果说 MapReduce 框架分为 Mapper 和 Reducer 两大模块，那么 Logstash 有仨

- Collect: 数据输入。对应 input
- Enrich: 数据处理。对应 filter
- Transport: 数据输出。对应 output


虽然模块仅仅比 MapReduce 框架多了一个，但是无三不成几，通过不同的拓扑结构，可以完成各类数据处理应用。不过这里我们主要还是以日志汇总处理系统的思路来进行介绍，一个典型的架构为：

![image](https://upload-images.jianshu.io/upload_images/79431-a6ffb52a2e4d5fdd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/893/format/webp)


## 配置语法
Logstash 设计了自己的 DSL —— 有点像 Puppet 的 DSL，或许因为都是用 Ruby 语言写的吧 —— 包括有区域，注释，数据类型(布尔值，字符串，数值，数组，哈希)，条件判断，字段引用等。

**区段(section)**
Logstash 用 {} 来定义区域。区域内可以包括插件区域定义，你可以在一个区域内定义多个插件。插件区域内则可以定义键值对设置。示例如下：

```
input {
    stdin {}
    syslog {}
}
```

**数据类型**
- bool
> debug => true

- string
> host => "hostname"

- number
> port => 514

- array
> match => ["datetime", "UNIX", "ISO8601"]

- hash
> options => {
    key1 => "value1",
    key2 => "value2"
}

**字段引用(field reference)**
如果你想在 Logstash 配置中使用字段的值，只需要把字段的名字写在中括号 [] 里就行了，这就叫字段引用。

对于 嵌套字段(也就是多维哈希表，或者叫哈希的哈希)，每层的字段名都写在 [] 里就可以了。比如，你可以从 geoip 里这样获取 longitude 值(是的，这是个笨办法，实际上有单独的字段专门存这个数据的)：

```
[geoip][location][0]
```

**条件判断(condition)**

Logstash从 1.3.0 版开始支持条件判断和表达式
表达式支持下面这些操作符：

- ==(等于), !=(不等于), <(小于), >(大于), <=(小于等于), >=(大于等于)
- =~(匹配正则), !~（不匹配正则）
- n(包含), not in(不包含)
- and(与), or(或), nand(非与), xor(非或)
- ()(复合表达式), !()(对复合表达式结果取反)

通常来说，你都会在表达式里用到字段引用。为了尽量展示全面各种表达式，下面虚拟一个示例：

```
if "_grokparsefailure" not in [tags] {
} else if [status] !~ /^2\d\d/ or ( [url] == "/noc.gif" nand [geoip][city] != "beijing" ) {
} else {
}
```

**命令行参数**
Logstash 提供了一个 shell 脚本叫 logstash 方便快速运行。它支持以下参数：

- -e

执行。我们在 "Hello World" 的时候已经用过这个参数了。事实上你可以不写任何具体配置，直接运行 

- --config 或 -f

文件。真实运用中，我们会写很长的配置，甚至可能超过 shell 所能支持的 1024 个字符长度。所以我们必把配置固化到文件里，然后通过 bin/logstash -f agent.conf 这样的形式来运行

此外，logstash 还提供一个方便我们规划和书写配置的小功能。你可以直接用 bin/logstash -f /etc/logstash.d/ 来运行。logstash 会自动读取 /etc/logstash.d/ 目录下所有 *.conf 的文本文件，然后在自己内存里拼接成一个完整的大配置文件，再去执行。

- --configtest 或 -t

测试。用来测试 Logstash 读取到的配置文件语法是否能正常解析

- --log 或 -l

日志。Logstash 默认输出日志到标准错误。生产环境下你可以通过 bin/logstash -l logs/logstash.log 命令来统一存储日志

- --pipeline-workers 或 -w

运行 filter 和 output 的 pipeline 线程数量。默认是 CPU 核数

- --pipeline-batch-size 或 -b

每个 Logstash pipeline 线程，在执行具体的 filter 和 output 函数之前，最多能累积的日志条数。默认是 125 条。越大性能越好，同样也会消耗越多的 JVM 内存

- --pipeline-batch-delay 或 -u

每个 Logstash pipeline 线程，在打包批量日志的时候，最多等待几毫秒。默认是 5 ms

- --pluginpath 或 -P

可以写自己的插件，然后用 bin/logstash --pluginpath /path/to/own/plugins 加载它们

- --verbose

输出一定的调试日志

- --debug

输出更多的调试日志

**配置文件**

从 Logstash 5.0 开始，新增了 $LS_HOME/config/logstash.yml 文件，可以将所有的命令行参数都通过 YAML 文件方式设置。同时为了反映命令行配置参数的层级关系，参数也都改成用.而不是-了


```
pipeline:
    workers: 24
    batch:
        size: 125
        delay: 5
```


## plugin的安装

从 logstash 1.5.0 版本开始，logstash 将所有的插件都独立拆分成 gem 包。这样，每个插件都可以独立更新，不用等待 logstash 自身做整体更新的时候才能使用了。

为了达到这个目标，logstash 配置了专门的 plugins 管理命令。


```
Usage:
    bin/logstash-plugin [OPTIONS] SUBCOMMAND [ARG] ...

Parameters:
    SUBCOMMAND                    subcommand
    [ARG] ...                     subcommand arguments

Subcommands:
    install                       Install a plugin
    uninstall                     Uninstall a plugin
    update                        Install a plugin
    list                          List all installed plugins

Options:
    -h, --help                    print help
```


## 运行方式

1. 标准的 service 方式

采用 RPM、DEB 发行包安装的读者，推荐采用这种方式

2. 最基础的 nohup 方式

想要维持一个长期后台运行的 logstash，你需要同时在命令前面加 nohup，后面加 &。

3. 更优雅的 SCREEN 方式（不详细说了）
4. 最推荐的 daemontools 方式（不详细说了）


## 插件配置
下面只列出列表，详细配置可查看官方文档

### 输入插件(Input)
- 读取文件(File)
- 标准输入(Stdin)
- 读取 Syslog 数据
- 读取网络数据(TCP)

### 编码插件(Codec)
Codec 是 logstash 从 1.3.0 版开始新引入的概念

在此之前，logstash 只支持纯文本形式输入，然后以过滤器处理它。但现在，我们可以在输入 期处理不同类型的数据，这全是因为有了 codec 设置

这里需要纠正之前的一个概念。Logstash 不只是一个input | filter | output 的数据流，而是一个 input | decode | filter | encode | output 的数据流！

codec 就是用来 decode、encode 事件的

codec 的引入，使得 logstash 可以更好更方便的与其他有自定义数据格式的运维产品共存，比如 graphite、fluent、netflow、collectd，以及使用 msgpack、json、edn 等通用数据格式的其他产品等

- json
- 合并多行数据(Multiline)
- collectd
- netflow

### 过滤器插件(Filter)
名为过滤器，其实提供的不单单是过滤的功能。在本章我们就会重点介绍几个插件，它们扩展了进入过滤器的原始数据，进行复杂的逻辑处理，甚至可以无中生有的添加新的 logstash 事件到后续的流程中去

- date
- grok
- dissect
- geoip
- json
- kv
- metrices
- mutate
- ruby
- split
- elapsed


### output配置
- elasticsearch
- email
- exec
- file
- nagios
- statsd
- stdout
- tcp
- hdfs

（本文完）
参考自：
- https://kibana.logstash.es/content/logstash/
- http://blog.csdn.net/wp500/article/details/41040213
- http://www.cnblogs.com/xing901022/p/4802822.html
- http://wdxtub.com/2016/07/24/logstash-guide/
- http://wdxtub.com/2016/07/26/elk-guide/
- http://yincheng.site/logstash




