---
title: RabbitMQ-理解消息通信-队列
tags: [RabbitMQ, 消息队列]
date: 2017-03-30 16:44:06
categories: RabbitMQ
link_title: rabbitMQ-queue
---
> AMQP消息路由必须有三部分：交换器、队列和绑定
<!--more-->

# 队列
AMQP消息路由必须有三部分：交换器、队列和绑定

生产者把消息发布到交换器上，消息最终到达队列，并被消费者接收，绑定决定了消息如何从路由器路由到特定的队列

消费者通过以下两种方式从特定的队列中接收消息：
- 通过AMQP的**basic.comsume**命令订阅，这样就会将信道置为接收模式，知道取消对队列的订阅为止
- 如果我们只想从队列获取单条消息而不是持续订阅，向队列发送单条消息是通过AMQP的**basic.get**，如果要获取更多的消息的话，需要再次发送basic.get命令。
（不要将basic.get放在循环里面来替代basic.consume）因为这样做会影响Rabbit的性能

**如果消息到达了无人订阅的队列呢？**
这种情况下，消息会在队列中等待，一旦有消费者订阅到该队列，那么队列上的消息会发给消费者

当Rabbit队列拥有多个消费者时，队列收到的消息将以循环（round-robin）的方式发给消费者，每条消息只会发送给一个订阅的消费者

消费者接收到每一条消息都必须进行确认，消费者必须通过AMQP的**basic.ack**命令显式的向RabbitMQ发送一个确认，或者在订阅到队列的时候就将**auto_ack**参数设置为true，当设置了auto_ack时，一旦消费者接收消息，RabbitMQ会自动视其确认了消息

注意：消费者对消息的确认和告诉生产者消息已经被接收了这两件事情毫无关系

消费者通过确认命令告诉rabbitMQ它已经正确的接收了消息，通过RabbitMQ才能安全地把消息从队列中删除

如果消费者收到一条消息，然后确认之前从Rabbit断开连接，RabbitMQ会认为这套消息没有分发，然后重新分发给下一个订阅的消费者

如果应用程序有bug而忘记确认消息的话，Rabbit将不会给该消费者发送更多的消息了，这是因为在上一条消息被确认之前，Rabbit会认为这个消费者并没有准备好接受下一条消息

**如果想要明确拒绝而不是确认收到该消息的话，该如何呢？**
两种选择：
1. 把消费者从RabbitMQ断开连接，这会导致RabbitMQ自动重新把消息入队给另一个消费者
2. 如果你使用的RabbitMQ 2.0.0或者更新的版本，那就使用AMQP的basic.reject命令，顾名思义：basic.reject允许消费者拒绝RabbitMQ发送的消息，如果把reject命令的requeue参数设置为true的话，RabbitMQ会将消息重新发送给下一个订阅的消费者

**为什么丢弃一条消息时，要使用basic.reject命令，并将requeue参数设置成false来替代确认消息呢？**
在将来的RabbitMQ版本中会支持一个特殊的“死信”队列，用来存放那些被拒绝而不重入队列的消息，如果应用程序想自动从死信队列功能总获益的话，需要使用reject命令。并将requeue参数设置为false

**如何创建队列**
消费者和生产者都能使用AMQP的**queue.declare**命令来创建队列

如果消费者在同一条信道上订阅了另一个队列的话，就无法再声明队列了。必须首先取消订阅，将信道设置为"传输"模式

创建队列时，你常常想要指定队列名称，如果不指定，Rabbit会分配一个随机名称并在queue,declare命令的响应中返回

以下是队列设置中另一些有用的参数：
- exclusive  
如果设置为true的话，队列将变成私有的，只有你的应用才能够消费队列消息
- auto-delete
当最后一个消费者取消订阅的时候，队列就会自动移除

**如果尝试声明一个已经存在的队列**
只要声明参数完全匹配现存的队列的话，Rabbit什么都不做，并成功返回

**如果你只想检验队列是否存在**，则可以设置queue,declare的passive选择为true，在该设置下，如果队列存在，那么queue.declare命令会成功返回，如果不存在，命令不会创建队列而会返回一个错误

# 总结
队列时AMQP消息通信的基础模块
- 为消息提供了住所，消息在此等待消费
- 对负载均衡来说，队列时绝佳方案，值只需附加一堆消费者，并让RabbitMQ以循环的方式均匀的分配发来的信息
- 队列时Rabbit中的消息的最后的终点


（注：内容整理自《RabbitMQ实战》）