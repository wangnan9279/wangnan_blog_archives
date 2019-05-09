---
title: 《Hadoop权威指南》书摘-关于Hive
link_title: hadoop-definitive-guide05
date: 2018-08-08 14:15:38
tags: [Hadoop]
categories: BigData
thumbnailImage: https://upload-images.jianshu.io/upload_images/79431-5177ccec636afe6b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/237/format/webp
thumbnailImagePosition: left
---
<span/>
<!-- more -->
![](https://upload-images.jianshu.io/upload_images/79431-5177ccec636afe6b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/237/format/webp)
<!-- toc -->

**转载请注明出处**
独立博客：http://wangnan.tech 
简书:http://www.jianshu.com/u/244399b1d776**
知乎：https://zhuanlan.zhihu.com/ghoststories

# 简介
hive是一个构建在hadoop上的数据仓框架

hive的设计目的是让精通SQL技能但java编程技能将对较弱的分析师能够对Fackbook存放在HDFS中的大规模数据集执行查询

# 安装
hive把sql查询转换为一系列hadoop集群上运行的作业

hive把数据组织为表，通过这种方式储存在hdfs上的数据赋予结构，元数据储存在metastore数据库中

安装过程很简单，必须在本地安装和集群上相同的版本的hadoop

下载一个hive的发布版本，解压，添加环境变量 启动hive shell

```
tar zxf apache-hive-x.y.z-bin.tar.gz
```

```
export HIVE_HOME = xxx
export PATH = $PATH:$HIVE_HOME/bin
```

```
hive
```

# hive shell 环境

hive shell 环境使我们和hive交互，发出hiveQL命令的主要方式，HiveQL是sql的一种方言

HiveSQL一般是大小写不敏感的，tab键会自动补全hive的关键字和函数

也可以非交互式模式允许hive的shell环境，使用-f 运行脚本
```
hive -f script.q
```
对于较短的脚本，可以用 -e 选项在行内嵌入命令
```
hive -e 'select * from xxx'
```

hive会默认把运行时信息打印输出到 standard error ，可以使用-S选项强制不显示这些消息

hive shell 其他特性：
使用！前缀来运行宿主操作系统命令
使用dfs命令来访问hadoop文件系统

# 运行hive
## 配置
hive配置文件为hive-site.xml在conf目录下

也可以传递 --config参数指定配置文件目录

配置文件中可以指定hadoop属性：fa-deaultFS和yarn.resourcemanager.adress，如果没有设定，默认值是使用本地文件系统和本地job runner

hive还允许用hive命令传递 --hiveconf选择来对单个会话设置属性

还可以在一个会话中使用SET命令更改设置

设置属性有一个优先级层次，从高到低
1. hive set命令
2. -hiveconf 选择
3. 配置文件中的值
4. hive默认值

**执行引擎**
hive的原始设计是以MapReduce作为执行引擎，目前还有包括执行引擎，Tez ,另外hive对spark的支持也在开发中，

tez和spark都是通用有向无环图DAG引擎，他们比MapReduce更加灵活，性能也更优越，比如，在使用MapReduce时，中间作业的输出会被物化储存到hdfs上，tez和spark则不同，他们可以根据hive规划器的请求，把中间结果写在本地磁盘，甚至是内存中缓存，以避免额外的复制开销

**metastore**
metastore是hive元数据的集中储存地，包括两部分：服务和后台数据的储存，默认情况下，metastore服务和hive服务运行在同一个JVM中，它包含一个内嵌的以本地磁盘作为储存的Derby数据库实例

如果要支持多会话，需要使用一个独立的数据库，这种配置称为本地metastore配置

对于独立的metastore,Mysql是一种受欢迎的选择

# hive与传统数据库相比

## 读时模式vs写时模式
在传统数据库中，表的模式是在加载数据时强制确定的，如果加载时不符合，则拒绝加载，这被称为 写时模式 schema on write

hive对数据的验证并不在加载数据时，而是在查询时，这被称为 读时模式 schema on read

读时模式可以使数据加载非常迅速，因为他不需要读取数据来进行解析

写时模式有利于提升查询性能

## 更新、事务、索引
直到现在，hive还没有考虑这些特性，因为hive被设计为用MapReduce操作HDFS数据，这样的环境下，全表扫描是常态操作

0.14.0b版本开始，hive可以对表做一些更细粒度的更新，也就是说，可以使用INSERT INTO TABLE ..VALUES 插入少量通过sql计算出来的值

0.7.0发布版本还引入了 表级和分区级 的锁，锁由zookeeper透明管理

hive索引分成两类：紧凑(compact)索引和位图(bitmap)索引

## 其他 sql-on-hadoop技术
- Impala
- Apache Drill
- Spark SQL

# HiveQL
注意点：
hive有四种复杂数据类型：ARRAY MAP STRUCT UNION

# 表
hive的表在逻辑上由储存的数据和描述表中数据形式的相关元数据组成

数据一般存放在hdfs中，但它也可以放在其他任何hadoop文件系统中，包括本地文件系统或者S3

HIVE把元数据存在关系型数据库中，而不是放在HDFS中

hive也提供了命名空间的支持，可以通过dbname.tablename完全限定一张表

## 托管表和外部表
hive创建表，默认情况是hive负责管理，这意味着hive把数据移入它的"仓库目录"

另一种选择是创建一个外部表（external table），这会让hive到仓库目录以外的位置访问数据

**区别**
两种表的区别表现在LOAD和DROP命令的语义上

托管表：LOAD 会把文件移动到表的仓库目录  DROP 后数据会扯到消失
外部表： hive知道数据并不是自己管理，因此不会把数据转移到自己的仓库目录，甚至，它不会检查这一外部位置是否存在 DROP外部表时，hive不会碰数据，只会删除元数据

如何选择？
作为一个经验法则，如果所有的处理都由hive完成，应该使用托管表
外部表可以用于从hive导出数据提供其他应用程序使用

## 分区和桶
hive把表组织成为分区（partition),这是一种根据分区列的值对表进行粗略划分的机制，使用分区可以加快数据分片的查询速度

表或者分区可以进一步分为桶（bucket），它会为数据提供额外的结构以获取更高效的查询处理

**分区**
- 一个表可以多个维度来进行分区
- 我们把数据加载到分区表的时候，要显示的指定分区值
- SHOW PARTITIONS 命令可以让hive告诉我们表中有哪些分区
- PARTITIONED BY 子句中的列定义是表中正式的列，成为分区列，但是，数据文件并不包含这些列的值，因为他们源于目录名

**桶**
- 桶可以是 取样或者说采样更高效
- 使用CLUSTERED BY 子句来指定划分桶的列和要划分的桶的个数
```
CREATE TABLE XXX(id int,name string)
CLUSTERED BY(ID) INTO 4 BUCKETS
```

- 桶中的数据可以根据一个或者多个列另外进行排序
- 物理上，每个桶就是表或者分区目录里的一个文件

## 储存格式
hive从两个维度对表进行管理，分别是行格式（row format）和文件格式（file format）

行格式指一行中的字段如何储存，按照hive的术语，行格式由定义serDe定义，serDe是“序列化和反序列化工具”(serializer-Deserializer)的合成词

文件格式指一行中字段容器的格式，最简单的格式是纯文本文件，也可以使用面向行的和面向列的二进制格式

**默认储存格式**
hive默认行内分隔符不是制表符，而是ASC2控制码集合中的Control-A,因为它出现在字段文本中的可能性比较小，在hive中无法对分隔符进行转义，因此，挑选一个不会在数据字段中用到的字符作为分隔符非常重要

集合类元素的默认分隔符是Control-B 
默认的映射建分隔符为字符control-C

**二进制储存格式：顺序文件，Avro数据文件，Parquet文件，RCFILE与ORCFile**

## 导入数据

- INSERT语句
```
INSERT OVERWRITE TABLE target
select col1,col2
from source
```

overwrite关键字意味着目标表的内容会被select语句的结果替换掉

- CREATE TABLE...AS SELECT 语句
可以把hive的查询的输出结果存放到一个新的表中

- 表的修改 
重命名表
```
ALTER TABLE source rename to target
```

添加新列
```
ALTER TABLE target ADD COLUMNS(col3 string)
```

## 表的丢弃

```
DROP table xxx
```


如果要删除表内所有数据，但是要保留表的定义，可以使用

```
TRUNCATE TABLE xxx
```


可以使用LIKE关键字创建一个与第一个表模式相同的新表

```
CREATE TABLE xxx LIKE xxx;
```

# 表的查询
## 排序和聚集
- ORDER BY
order by将对输入执行并行全排序

- SORT BY
很多情况，并不需要结果是全局排序的，sort by为每个reducer产生一个排序文件

- DISTRIBUTE BY
在有些情况下，你需要控制某个特定行应该到哪个reducer，目的是为了进行后续的聚集操作

## MapReduce脚本
和Hadoop Streaming类似，TRANSFORM MAP REDUCE 子句可以在hive中调用外部脚本或程序

比如写了一下脚本 xxx.py

然后sql语句


```
ADD FILE /xxx/xxx /xxx/xxx.py
```

```
SELECT TRANSFORM(a,b,c) USING 'xxx.py' AS year,temperature
```

## 连接
- 内连接 JOIN（俩边都匹配上的才会是一行）
- 外连接 OUTER JOIN（只要有一边有就可以）
- 半连接 （右表只能在on子句中出现）
- map连接（可以利用分桶的表，作用于左侧表的桶的mapper加载右侧表中对应的桶即可执行连接）

## 视图
- 视图是select语句定义的虚表
- 在hive中，创建视图时并不把视图物化到硬盘，视图的select语句只是在执行引用视图的语句时才执行

# 用户自定义函数
你要写的查询有时候无法轻松或者不能使用hive提供的内置函数表示，通过编写用户自定义函数（user-defined function UDF）hive可以方便地插入用户写的处理代码并在查询中调用他们

UDF必须用Java语言编写，Hive本身就是java写的

UDF有三种：普通UDF,用户自己聚集函数（UDAF）,用户定义表生成函数（UDTF）
- UDF 操作用于与单个数据行，且产生一个数据行作为输出
- UDAF 接受多个输入数据行，并产生一个输出数据行
- UDTF 操作作用于单个数据行，且产生多个数据行（即一个表）作为 输出

## 写UDF
- 一个UDF必须是org.apache.hadoop.hive.ql.exec.UDF的子类
- 一个UDF必须至少实现了evaluate()方法
- hive支持在udf是使用java基本类型
- 为了在hive中使用UDF,我们 需要把变异后的java类打包成一个jar文件，然后在metastore中注册这个函数并使用CREATE FUNCTION语句为它起名
- 如果是集群上，则应该将JAR文件复制到HDFS
- UDF对大小写不敏感
- 利用TEMPORARY 关键字可以创建一个仅在hive会话期间有效的函数，也就是说这个函数 没有在metastore中持久化储存

# 写UDAF
必须实现下面五个方法
- init()
- iterate()
- terminatePartial()
- merge()
- terminate()