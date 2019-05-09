---
title: Java-ThreadPoolExecutor
link_title: java-ThreadPoolExecutor
date: 2017-07-25 16:14:51
tags: [Java, JavaConcurrent]
categories: Java
thumbnailImage: https://upload-images.jianshu.io/upload_images/79431-4c5250ccfeea7489.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/509/format/webp	
thumbnailImagePosition: left
---
<span/>
<!-- more -->
![](https://upload-images.jianshu.io/upload_images/79431-4c5250ccfeea7489.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/509/format/webp)
<!-- toc -->
# 概述
从Java5开始，Java提供了自己的线程池。每次只执行指定数量的线程，java.util.concurrent.ThreadPoolExecutor 就是这样的线程池

# 构造函数
```java
public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue,RejectedExecutionHandler handler);  
```
## 参数介绍
### corePoolSize
核心线程数，指保留的线程池大小（不超过maximumPoolSize值时，线程池中最多有corePoolSize 个线程工作）。 

### maximumPoolSize 
指的是线程池的最大大小（线程池中最大有corePoolSize 个线程可运行）。

### keepAliveTime 
指的是空闲线程结束的超时时间（当一个线程不工作时，过keepAliveTime 长时间将停止该线程）。

### workQueue 
表示存放任务的队列（存放需要被线程池执行的线程队列）。 

## 拒绝策略（添加任务失败后如何处理该任务)
1. 线程池刚创建时，里面没有一个线程。任务队列是作为参数传进来的。不过，就算队列里面有任务，线程池也不会马上执行它们。
2.当调用 execute() 方法添加一个任务时，线程池会做如下判断：
2.1. 如果正在运行的线程数量**小于 corePoolSize**，那么马上创建线程运行这个任务；
2.2. 如果正在运行的线程数量**大于或等于 corePoolSize**，那么将这个任务放入队列。
2.3. **如果这时候队列满了，而且正在运行的线程数量小于 maximumPoolSize**，那么还是要创建线程运行这个任务；
2.4. **如果队列满了，而且正在运行的线程数量大于或等于 maximumPoolSize**，那么线程池会抛出异常，告诉调用者“我不能再接受任务了”。
3. 当一个线程完成任务时，它会从队列中取下一个任务来执行。
4. **当一个线程无事可做，超过一定的时间（keepAliveTime）时，线程池会判断，如果当前运行的线程数大于 corePoolSize，那么这个线程就被停掉。所以线程池的所有任务完成后，它最终会收缩到 corePoolSize 的大小**。

###  说明
- 这个过程说明，并不是先加入任务就一定会先执行。
- 假设队列大小为 4，corePoolSize为2，maximumPoolSize为6，那么当加入15个任务时，执行的顺序类似这样：首先执行任务 1、2，然后任务3~6被放入队列。这时候队列满了，任务7、8、9、10 会被马上执行，而任务 11~15 则会抛出异常。最终顺序是：1、2、7、8、9、10、3、4、5、6。
- 当然这个过程是针对指定大小的ArrayBlockingQueue<Runnable>来说，如果是**LinkedBlockingQueue<Runnable>**，因为该队列无大小限制，所以不存在上述问题。

## 三种方法将线程添加到线程队列
对于 java.util.concurrent.BlockingQueue 类有有三种方法将线程添加到线程队列里面，然而如何区别三种方法的不同呢，其实在队列未满的情况下结果相同，都是将线程添加到线程队列里面，区分就在于当线程队列已经满的时候，此时
- public boolean add(E e) 方法将抛出IllegalStateException异常，说明队列已满。
- public boolean offer(E e) 方法则不会抛异常，只会返回boolean值，告诉你添加成功与否，队列已满，当然返回false。
- public void put(E e) throws InterruptedException 方法则一直阻塞（即等待，直到线程池中有线程运行完毕，可以加入队列为止）。

![](Java-ThreadPoolExecutor/01.png)

## 线程池的排除策略
1. 如果运行的线程少于 corePoolSize，则 Executor 始终首选添加新的线程，而不进行排队。
2. 如果运行的线程等于或多于 corePoolSize，则 Executor 始终首选将请求加入队列，而不添加新的线程。
3. 如果无法将请求加入队列，则创建新的线程，除非创建此线程超出 maximumPoolSize，在这种情况下，任务将被拒绝。

# 总结
1. 线程池可立即运行的最大线程数 即maximumPoolSize 参数。
2. 线程池能包含的最大线程数 = 可立即运行的最大线程数 + 线程队列大小 (一部分立即运行，一部分装队列里等待)
3. 核心线程数可理解为建议值，即建议使用的线程数，或者依据CPU核数
4. add，offer，put三种添加线程到队列的方法只在队列满的时候有区别，add为抛异常，offer返回boolean值，put直到添加成功为止。
5.同理remove，poll， take三种移除队列中线程的方法只在队列为空的时候有区别， remove为抛异常，poll为返回boolean值， take等待直到有线程可以被移除。

参考：
- http://blog.csdn.net/shixing_11/article/details/7109471


