---
title: Java多线程简介之休眠、优先级、让步等
date: 2017-06-13 14:36:33
tags: [Java,JavaConcurrent]
categories: Java
link_title: java-concurrent02
thumbnailImage: https://i.loli.net/2019/09/24/4kPOL3i5aAqcfNs.png
thumbnailImagePosition: left
---
<!-- toc -->
<!-- more -->
![](https://i.loli.net/2019/09/24/4kPOL3i5aAqcfNs.png)
# 休眠

影响任务的一种简单方式是调用sleep（），这将使任务中止执行给定的时间。 
<!-- more -->

例程：
```java
//: concurrency/SleepingTask.java
// Calling sleep() to pause for a while.
import java.util.concurrent.*;

public class SleepingTask extends LiftOff {
  public void run() {
    try {
      while(countDown-- > 0) {
        System.out.print(status());
        // Old-style:
        // Thread.sleep(100);
        // Java SE5/6-style:
        TimeUnit.MILLISECONDS.sleep(100);
      }
    } catch(InterruptedException e) {
      System.err.println("Interrupted");
    }
  }
  public static void main(String[] args) {
    ExecutorService exec = Executors.newCachedThreadPool();
    for(int i = 0; i < 5; i++)
      exec.execute(new SleepingTask());
    exec.shutdown();
  }
} /* Output:
#0(9), #1(9), #2(9), #3(9), #4(9), #0(8), #1(8), #2(8), #3(8), #4(8), #0(7), #1(7), #2(7), #3(7), #4(7), #0(6), #1(6), #2(6), #3(6), #4(6), #0(5), #1(5), #2(5), #3(5), #4(5), #0(4), #1(4), #2(4), #3(4), #4(4), #0(3), #1(3), #2(3), #3(3), #4(3), #0(2), #1(2), #2(2), #3(2), #4(2), #0(1), #1(1), #2(1), #3(1), #4(1), #0(Liftoff!), #1(Liftoff!), #2(Liftoff!), #3(Liftoff!), #4(Liftoff!),
*///:~
```
对sleep的调用可以抛出InterruptedException异常，并且你可以看到，它在main()中被捕获，因为异常不能跨越线程传播回main()，所以你必须在本地处理所有在任务内部产生的异常。


# 优先级
线程的优先级先线程的重要性传递给调度器，尽管cpu处理现有线程集的顺序是不确定的，但是调度器更倾向于让优先级更高的线程先执行。线程优先级较低的线程不是不执行，仅仅是执行的频率较低。

getPriority()读取现有的优先级 
setPriority()来修改它

例程：
```java
//: concurrency/SimplePriorities.java
// Shows the use of thread priorities.
import java.util.concurrent.*;

public class SimplePriorities implements Runnable {
  private int countDown = 5;
  private volatile double d; // No optimization
  private int priority;
  public SimplePriorities(int priority) {
    this.priority = priority;
  }
  public String toString() {
    return Thread.currentThread() + ": " + countDown;
  }
  public void run() {
    Thread.currentThread().setPriority(priority);
    while(true) {
      // An expensive, interruptable operation:
      for(int i = 1; i < 100000; i++) {
        d += (Math.PI + Math.E) / (double)i;
        if(i % 1000 == 0)
          Thread.yield();
      }
      System.out.println(this);
      if(--countDown == 0) return;
    }
  }
  public static void main(String[] args) {
    ExecutorService exec = Executors.newCachedThreadPool();
    for(int i = 0; i < 5; i++)
      exec.execute(
        new SimplePriorities(Thread.MIN_PRIORITY));
    exec.execute(
        new SimplePriorities(Thread.MAX_PRIORITY));
    exec.shutdown();
  }
} /* Output: (70% match)
Thread[pool-1-thread-6,10,main]: 5
Thread[pool-1-thread-6,10,main]: 4
Thread[pool-1-thread-6,10,main]: 3
Thread[pool-1-thread-6,10,main]: 2
Thread[pool-1-thread-6,10,main]: 1
Thread[pool-1-thread-3,1,main]: 5
Thread[pool-1-thread-2,1,main]: 5
Thread[pool-1-thread-1,1,main]: 5
Thread[pool-1-thread-5,1,main]: 5
Thread[pool-1-thread-4,1,main]: 5
...
*///:~
```

通过Thread.currentThread()来获取对驱动该任务的Thread对象的引用

尽管JDK有10个优先级，但它与多数操作系统都不能映射得很好，唯一可移植的方法是当调整优先级的时候。只使用 MAX_PRIORITY NORM_PRIORITY MIN_PRIORITY 三种级别

# 让步
如果知道已经完成了在run()方法的循环的一次迭代过程中所需的工作，就可以给线程调度机制一个暗示：你的工作已近做的差不多了，可以让别的线程使用CPU了，这个暗示将通过调用yield()方法作出（不过这只是一个暗示，没有任何机制保证它将被采纳），当调用yield()时，你也是在建议具有相同优先级的其他线程可以运行

# 后台线程
后台线程(deamon)线程，又叫守护线程，是指程序运行的时候在后台提供的一种通用服务线程。这种线程不属于程序中不可或缺的部分，因此，当所有非后台线程结束时，程序也就终止了，同时会杀死进程中的所有后台线程

必须在线程启动调用setDeamon()方法，才能把它设置为后台线程

例程：
```java
//: concurrency/SimpleDaemons.java
// Daemon threads don't prevent the program from ending.
import java.util.concurrent.*;
import static net.mindview.util.Print.*;

public class SimpleDaemons implements Runnable {
  public void run() {
    try {
      while(true) {
        TimeUnit.MILLISECONDS.sleep(100);
        print(Thread.currentThread() + " " + this);
      }
    } catch(InterruptedException e) {
      print("sleep() interrupted");
    }
  }
  public static void main(String[] args) throws Exception {
    for(int i = 0; i < 10; i++) {
      Thread daemon = new Thread(new SimpleDaemons());
      daemon.setDaemon(true); // Must call before start()
      daemon.start();
    }
    print("All daemons started");
    TimeUnit.MILLISECONDS.sleep(175);
  }
} /* Output: (Sample)
All daemons started
Thread[Thread-0,5,main] SimpleDaemons@530daa
Thread[Thread-1,5,main] SimpleDaemons@a62fc3
Thread[Thread-2,5,main] SimpleDaemons@89ae9e
Thread[Thread-3,5,main] SimpleDaemons@1270b73
Thread[Thread-4,5,main] SimpleDaemons@60aeb0
Thread[Thread-5,5,main] SimpleDaemons@16caf43
Thread[Thread-6,5,main] SimpleDaemons@66848c
Thread[Thread-7,5,main] SimpleDaemons@8813f2
Thread[Thread-8,5,main] SimpleDaemons@1d58aae
Thread[Thread-9,5,main] SimpleDaemons@83cc67
...
*///:~
```

可以调用isDeamon()方法来确定线程是否是一个后台线程，如果是一个后台线程，那么它创建的任何线程将被自动设置成后台线程

当最后一个非后台线程终止时，后台线程会“突然”终止。

# 加入一个线程
一个线程在其他线程上调用join()方法，其效果是等待一段时间直到第二个线程结束才继续执行，如果某个线程在另一个线程t上调用t.join()，此线程将被挂起。直到目标线程t结束才恢复

也可以在调用join()时带上一个超时参数，这样如果目标线程在这段时间到期时还没有结束的话，join()方法总能返回。

对join()方法的调用可以被中断，做法是在线程上调用interrupt()方法，这需要用到try-catch子句

# 异常捕获
由于线程的本质特性，使得你不能捕获从线程中逃逸的异常，一旦异常逃逸任务的run()方法，他就会向外传播到控制台，除非你采取特殊的步骤捕获这种错误的异常

Thread.UncaughtExceptionHandler是Java SE5中的新接口 ，它允许你在每个Thread对象上附着一个异常处理器

Thread.UncaughtExceptionHandler的uncaughtException()方法会在线程因未捕获的异常而临近死亡时被调用。

示例代码：
```java
//: concurrency/CaptureUncaughtException.java
import java.util.concurrent.*;

class ExceptionThread2 implements Runnable {
  public void run() {
    Thread t = Thread.currentThread();
    System.out.println("run() by " + t);
    System.out.println(
      "eh = " + t.getUncaughtExceptionHandler());
    throw new RuntimeException();
  }
}

class MyUncaughtExceptionHandler implements
Thread.UncaughtExceptionHandler {
  public void uncaughtException(Thread t, Throwable e) {
    System.out.println("caught " + e);
  }
}

class HandlerThreadFactory implements ThreadFactory {
  public Thread newThread(Runnable r) {
    System.out.println(this + " creating new Thread");
    Thread t = new Thread(r);
    System.out.println("created " + t);
    t.setUncaughtExceptionHandler(
      new MyUncaughtExceptionHandler());
    System.out.println(
      "eh = " + t.getUncaughtExceptionHandler());
    return t;
  }
}

public class CaptureUncaughtException {
  public static void main(String[] args) {
    ExecutorService exec = Executors.newCachedThreadPool(
      new HandlerThreadFactory());
    exec.execute(new ExceptionThread2());
  }
} /* Output: (90% match)
HandlerThreadFactory@de6ced creating new Thread
created Thread[Thread-0,5,main]
eh = MyUncaughtExceptionHandler@1fb8ee3
run() by Thread[Thread-0,5,main]
eh = MyUncaughtExceptionHandler@1fb8ee3
caught java.lang.RuntimeException
*///:~
```

# 共享受限资源
对于并发工作，你需要某种方式来防止两个任务访问相同的资源，至少关键阶段不能出现这种现象

基本上所有的并发模式在解决线程冲突问题的时候，都是采取序列化访问共享资源的方案

## synchronized
Java以提供关键字synchronized的形式，为防止资源冲突提供了内置支持 
要控制对共享资源的访问，得先把它包装进一个对象

## Lock
Lock对象必须显示的创建、锁定和释放。与synchronized的形式的相比，代码缺乏优雅性。但是，对于解决某些类型的问题来说。它更加灵活。

如果使用synchronized关键字，某些事物失败了，那么就会抛出一个异常。但是你没有机会去做任何清理工作。以维护系统使其处于良好状态。有了显示的Lock对象，你就可以使用finally子句将系统维护在正常的状态了。

## volatilez
JVM可以将64位（long和double变量）读取和写入当做两个分离的32来执行，这就产生了一个在读取和写入操作中间发生上下文切换，从而导致不同任务可以看到不正确的结果的可能性（这有时被称为字撕裂）

当定义long和double变量时，如果使用volatile关键字，就会获取原子性

如果一个域完全由synchronized方法或语句块来保护，那就不必将其设置为是volatile的

当一个域的值依赖于它之前的值时，volatile就无法工作了。如果某个域的值受到其他域的值的限制。那么volatile也无法工作。使用volatile而不是synchronized的唯一安全的情况是类中只有一个可变的域。

## 同步控制块
有时我们只希望防止多个线程同时访问方法内部的部分代码而不是防止访问整个方法，通过这种方式分离出来的代码段被称为临界区，她也使用synchronized关键字，这里synchronized被用来指定某对象，此对象的锁被用来对花括号内的代码进行同步控制

```java
synchronized(syncObject){

}
```

在进入此段代码前，必须得到syncObject对象的锁。如果其他线程也已经得到这个锁，那么就要等到锁被释放以后，才能进入临界区。

使用它的好处是，可以使多个任务访问对象的时间性能得到显著提高

# ThreadLocal
防止任务在共享资源上产生冲突的第二种方式是根除对变量的共享，线程本地是一种自动化机制，可以使用相同变量的每个不同线程都创建不同的存储，

get()方法将返回与其线程相关联的对象的副本，而set()会将参数数据插入到为其线程储存的对象中。


（注：内容整理自《Thinking in Java》）

