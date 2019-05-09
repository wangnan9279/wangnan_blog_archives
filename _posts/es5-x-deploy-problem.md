---
title: ES5.x部署遇到的问题汇总
link_title: es5.x-deploy-problem
date: 2017-09-08 15:12:24
tags: [ElasticSearch]
categories: ElasticSearch
thumbnailImage: https://upload-images.jianshu.io/upload_images/79431-a021cbc24f0ba70f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/509/format/webp	
thumbnailImagePosition: left
---
<span/>
<!-- more -->
![](https://upload-images.jianshu.io/upload_images/79431-a021cbc24f0ba70f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/509/format/webp)
<!-- toc -->
# 问题一
> can not run elasticsearch as root

不能以root用户启动ES服务器

非要以root用户运行,对于5.X，在config/jvm.options配置文件中，添加
> -Des.insecure.allow.root=true

# 问题二
> max virtual memory areas vm.max_map_count [65530] likely too low, increase to at least [262144]

原因：

最大虚拟内存太小 

解决：

1. 切换到root用户
2. vi /etc/sysctl.conf
3. 添加内容：vm.max_map_count=655360
4. 执行命令：sysctl -p

# 问题三
> max number of threads [1024] for user [xxx] likely too low, increase to at least [2048]

原因：

无法创建本地线程问题,用户最大可创建线程数太小 

解决：

1. 切换到root用户
2. vi /etc/security/limits.d/90-nproc.conf
3. 找到如下内容：
```xml
* soft nproc 1024
```
修改为
```xml
* soft nproc 2048
```
保存、退出、重新登录才可生效

# 问题四
> max file descriptors [4096] for elasticsearch process likely too low, increase to at least [65536]

原因：

无法创建本地文件问题,用户最大可创建文件数太小

解决方案： 
1. 切换到root用户
2. vi /etc/security/limits.conf
3. 添加如下内容:
```xml
*  soft nofile 65536

* hard nofile 131072

* soft nproc 2048

* hard nproc 4096
```
*表示所有用户

保存、退出、重新登录才可生效

# 问题五
> system call filters failed to install; check the logs and fix your configuration or disable system call filters at your own risk

这是在因为操作系统不支持SecComp，而ES5.2.2默认bootstrap.system_call_filter为true进行检测，所以导致检测失败，失败后直接导致ES不能启动。

在elasticsearch.yml中配置bootstrap.system_call_filter为false，注意要在Memory下面:
```xml
bootstrap.memory_lock: false
bootstrap.system_call_filter: false
```

# 问题六
```xml
[2016-11-06T16:27:21,712][WARN ][o.e.b.JNANatives ] unable to install syscall filter:

Java.lang.UnsupportedOperationException: seccomp unavailable: requires kernel 3.5+ with CONFIG_SECCOMPandCONFIG_SECCOMP_FILTERcompiledinatorg.elasticsearch.bootstrap.Seccomp.linuxImpl(Seccomp.java:349) ~[elasticsearch-5.0.0.jar:5.0.0]

at org.elasticsearch.bootstrap.Seccomp.init(Seccomp.java:630) ~[elasticsearch-5.0.0.jar:5.0.0]
```

报了一大串错误，大家不必惊慌，其实只是一个警告，主要是因为你Linux版本过低造成的。

1、重新安装新版本的Linux系统 
2、警告不影响使用，可以忽略

# 问题七
> org.elasticsearch.transport.RemoteTransportException: Failed to deserialize exception response from stream

原因:

ElasticSearch节点之间的jdk版本不一致

解决方案：

ElasticSearch集群统一jdk环境

# 问题八
> Unsupported major.minor version 52.0

原因:

jdk版本问题太低 

解决方案：

更换jdk版本，ElasticSearch5.0.0支持jdk1.8.0