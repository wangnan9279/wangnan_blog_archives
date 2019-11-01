---
title: 高并发系统限流设计
link_title: current-limiting
date: 2017-10-25 10:07:58
tags: []
categories: 架构
thumbnailImage: https://i.loli.net/2019/09/24/IlPnH2SJFTa1WjD.png	
thumbnailImagePosition: left
---
<span/>
<!-- more -->
![](https://i.loli.net/2019/09/24/IlPnH2SJFTa1WjD.png)
<!-- toc -->
# 概述
高并发系统时有三把利器用来保护系统：缓存、降级和限流，缓存的目的是提升系统访问速度和增大系统能处理的容量，降级是当服务出问题或者影响到核心流程的性能则需要暂时屏蔽掉，待高峰或者问题解决后再打开，而有些场景并不能用缓存和降级来解决，比如稀缺资源（秒杀、抢购）、写服务（如评论、下单）、频繁的复杂查询（评论的最后几页），因此需有一种手段来限制这些场景的并发/请求量，即限流

限流的目的是通过对并发访问/请求进行限速或者一个时间窗口内的的请求进行限速来保护系统，一旦达到限制速率则可以拒绝服务（定向到错误页或告知资源没有了）、排队或等待（比如秒杀、评论、下单）、降级（返回托底数据或默认数据，如商品详情页库存默认有货）。

## 限流方式
- 限制总并发数（比如数据库连接池、线程池）
- 限制瞬时并发数（如nginx的limit_conn模块，用来限制瞬时并发连接数）
- 限制时间窗口内的平均速率（如Guava的RateLimiter、nginx的limit_req模块，限制每秒的平均速率）
- 限制远程接口调用速率
- 限制MQ的消费速率。
- 可以根据网络连接数、网络流量、CPU或内存负载等来限流

# 限流算法
## 令牌桶
![](https://upload-images.jianshu.io/upload_images/79431-d19d892c42a49a52.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/485/format/webp)
## 漏桶
![](https://upload-images.jianshu.io/upload_images/79431-03d86f2392f16acd.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/509/format/webp)
## 计数器
有时候我们还使用计数器来进行限流，主要用来限制总并发数，比如数据库连接池、线程池、秒杀的并发数；只要全局总请求数或者一定时间段的总请求数设定的阀值则进行限流，是简单粗暴的总数量限流，而不是平均速率限流

## 令牌桶vs漏桶
令牌桶限制的是平均流入速率，允许突发请求，并允许一定程度突发流量
漏桶限制的是常量流出速率，从而平滑突发流入速率

# 应用级别限流
## 限流总并发/连接/请求数
如果你使用过Tomcat，其Connector其中一种配置有如下几个参数:
- acceptCount:如果Tomcat的线程都忙于响应，新来的连接会进入队列排队，如果超出排队大小，则拒绝连接
- maxConnections:瞬时最大连接数，超出的会排队等待
- maxThreads:Tomcat能启动用来处理请求的最大线程数，如果请求处理量一直远远大于最大线程数则可能会僵死

## 限流总资源数 
可以使用池化技术来限制总资源数：连接池、线程池。比如分配给每个应用的数据库连接是100，那么本应用最多可以使用100个资源，超出了可以等待或者抛异常

## 限流某个接口的总并发/请求数
可以使用Java中的AtomicLong，示意代码：
```java
try{
if(atomic.incrementAndGet() > 限流数) {
//拒绝请求
}
//处理请求
}finally{
atomic.decrementAndGet();
}

```

## 限流某个接口的时间窗请求数
使用Guava的Cache,示意代码：
```java
LoadingCache counter =
CacheBuilder.newBuilder()
.expireAfterWrite(2, TimeUnit.SECONDS)
.build(newCacheLoader() {
@Override
publicAtomicLong load(Long seconds)throwsException {
return newAtomicLong(0);
}
});
longlimit =1000;
while(true) {
//得到当前秒longcurrentSeconds = System.currentTimeMillis() /1000;
if(counter.get(currentSeconds).incrementAndGet() > limit) {
System.out.println("限流了:"+ currentSeconds);
continue;
}
//业务处理
}

```

## 平滑限流某个接口的请求数
之前的限流方式都不能很好地应对突发请求，即瞬间请求可能都被允许从而导致一些问题；因此在一些场景中需要对突发请求进行整形，整形为平均速率请求处理

Guava框架提供了令牌桶算法实现

Guava RateLimiter提供了令牌桶算法实现：平滑突发限流(SmoothBursty)和平滑预热限流(SmoothWarmingUp)实现

**平滑突发限流(SmoothBursty)**
```java
RateLimiter limiter = RateLimiter.create(5);
System.out.println(limiter.acquire());
System.out.println(limiter.acquire());
System.out.println(limiter.acquire());
System.out.println(limiter.acquire());
System.out.println(limiter.acquire());
System.out.println(limiter.acquire());
将得到类似如下的输出：
0.0
0.198239
0.196083
0.200609
0.199599
0.19961

```
limiter.acquire(5)表示桶的容量为5且每秒新增5个令牌



**平滑预热限流(SmoothWarmingUp)**
```java
RateLimiter limiter = RateLimiter.create(5,1000, TimeUnit.MILLISECONDS);
for(inti =1; i <5;i++) {
System.out.println(limiter.acquire());
}
Thread.sleep(1000L);
for(inti =1; i <5;i++) {
System.out.println(limiter.acquire());
}
将得到类似如下的输出：
0.0
0.51767
0.357814
0.219992
0.199984
0.0
0.360826
0.220166
0.199723
0.199555

```

SmoothWarmingUp创建方式：RateLimiter.create(doublepermitsPerSecond, long warmupPeriod, TimeUnit unit)
permitsPerSecond表示每秒新增的令牌数，warmupPeriod表示在从冷启动速率过渡到平均速率的时间间隔。

速率是梯形上升速率的，也就是说冷启动时会以一个比较大的速率慢慢到平均速率；然后趋于平均速率（梯形下降到平均速率）。可以通过调节warmupPeriod参数实现一开始就是平滑固定速率。


# 分布式限流
分布式限流最关键的是要将限流服务做成原子化,而解决方案可以使使用redis+lua或者nginx+lua技术进行实现

详见参考文章

# 接入层限流

接入层通常指请求流量的入口，该层的主要目的有：负载均衡、非法请求过滤、请求聚合、缓存、降级、限流、A/B测试、服务质量监控等等

对于Nginx接入层限流可以使用Nginx自带了两个模块：连接数限流模块ngx_http_limit_conn_module和漏桶算法实现的请求限流模块ngx_http_limit_req_module。还可以使用OpenResty提供的Lua限流模块lua-resty-limit-traffic进行更复杂的限流场景。

limit_conn用来对某个KEY对应的总的网络连接数进行限流，可以按照如IP、域名维度进行限流。

limit_req用来对某个KEY对应的请求的平均速率进行限流，并有两种用法：平滑模式（delay）和允许突发模式(nodelay)。

OpenResty提供的Lua限流模块lua-resty-limit-traffic进行更复杂的限流场景

详见参考文章

（本文完）

# 参考自文章
- http://www.jianshu.com/p/0d7ca597ebd2
- http://www.jianshu.com/p/b7945524a37b
- http://www.54tianzhisheng.cn/2017/09/23/Guava-limit/
