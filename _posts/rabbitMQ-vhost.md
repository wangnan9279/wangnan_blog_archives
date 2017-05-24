---
title: RabbitMQ-理解消息通信-虚拟主机和隔离
tags: [RabbitMQ, 消息队列]
date: 2017-04-01 10:35:48
categories: RabbitMQ
link_title: rabbitMQ-vhost
---
![](http://onxkn9cbz.bkt.clouddn.com/rabbitmq.png)

> 每个RabbitMQ服务器都能创建虚拟的消息服务器，我们称之为**虚拟主机(vhost)**每一个vhost本质上是一个mini版的RabbitMQ服务器，拥有自己的队列、交换器和绑定等等


# 多租户模式：虚拟主机和隔离

## 概述
- 每个RabbitMQ服务器都能创建虚拟的消息服务器，我们称之为**虚拟主机(vhost)**每一个vhost本质上是一个mini版的RabbitMQ服务器，拥有自己的队列、交换器和绑定等等
- 更重要的是，**他拥有自己的权限机制**，**这使得你能够安全地使用一个RabbitMQ服务器来服务众多的应用程序**
- vhost就像是虚拟机之与物理服务器一样：他们在各个实例间提供逻辑上的分离，允许你为不同程序安全保密地运行数据，它既能将同一个Rabbit的众多客户区分开来，又可以避免队列和交换器命名冲突
- vhost是AMQP概念的基础，你必须在连接时进行指定
- RabbitMQ包含了一个开箱即用的默认vhost:"/"，如果你不需要多个vhost，那么就使用默认的吧，使用缺省的guest用户名和密码guest就可以访问默认的vhost
- 当你在RabbitMQ集群上创建vhost，整个集群上都会创建该vhost，vhost不仅消除了为基础架构中的每一层运行一个RabbitMQ服务器的需要，同样也避免了为每一层创建不同集群

## 如何创建vhost
vhost和权限控制非常独特，他们是AMQP中唯一无法通过AMQP协议的基元（不同与队列，交换器和绑定）

**创建vhost**

你需要通过RabbitMQ的安装路径下的./sbin/目录中的**rabbitmqctl**工具来创建

运行：

```
rabbitmqctl add_vhost[vhost_name]
```
可以创建一个vhost，其中[vhost_name]就是你想要创建的vhost

**删除vhost**

```
rabbitmqctl delete_vhost[vhsost_name]
```
**查看Rabbit服务器上运行着那些vhost**

```
rabbitmqctl list_vhost
```
你就会看到如下所示的内容

```
$ ./sbin/rabbitmqctl list_vhosts
Listing vhosts ...
/
oak
sycamore
...done.
```

**管理远程RabbitMQ节点**

```
-n rabbit@[server_name]
```
rabbit表示Erlang应用程序名称
[server_name]表示ip

（注：内容整理自《RabbitMQ实战》）











