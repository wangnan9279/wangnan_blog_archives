---
title: Java多线程简介之基本概念、Thread类、Executor
date: 2017-03-16 14:19:15
tags: [Java, 并发, 多线程]
categories: Java
link_title: java-concurrent01
---
>在没有接触并发编程概念之前，你学到的都是有关顺序编程的知识，即程序中的所以事物在任意时刻都只能执行一个步骤
并行编程可以使程序执行速度得到极大的提高，或者为设计某些类型的程序提供更易用的模型，或者两者皆有。当并行执行的任务彼此开始产生互相干涉时，实际的并发问题就会接踵而至。这些并发问题，如果视而不见，就会遭到其反噬。
因此，使用并发时你得自食其力，并且只有变得多疑而且自信，才能用Java编写出可靠的多线程代码。

<!-- more -->

# 并发的多面性

用并发解决的问题大体上可以分为“速度”和“设计可管理性”两种

如果程序中的某个任务因为改程序控制范围外的某些条件（通常是I/O）而导致不能继续执行，那么我们就说这个任务或线程阻塞了，如果没有并发，则整个程序都将停止下来，直至外部条件发生变化。但是，如果使用并发来编写程序，那么当一个任务发生阻塞时，程序中的其他任务还可以继续执行。因此这个任务可以保持继续向前执行。

实现并发最直接的方式就是操作系统级别的使用进程。进程是运行在它自己的地址空间内的自包容程序。

因此编写多线程程序最基本的困难在于协调不同线程驱动任务之间对这些资源的使用，以使得这些资源不会被多个任务访问。

某些编程语言被设计为可以将并发任务彼此隔离，这些语言通常为称为函数式语言，其中每个函数调用都不会产生任何副作用（并因此而不能干涉其他函数），并因此可以当作独立的任务驱动。

java的线程机制是抢占式的，这表示调度机会周期性地中断线程，将上下文切换到另一个线程，从而为每个线程都提供时间片，使得每个线程都会分配到数量合适的时间去驱动它的任务。在协作式系统中，每个任务都会自动的放弃控制，这要求程序员要有意识地在每个任务中插入某种类型的让步语句

# 基本的线程机制

一个线程就是在进程中一个单一的顺序控制流，因此，单个进程可以拥有多个并发执行任务

# 定义任务

要想定义一个任务，只需要实现Runnable接口并编写run()方法

例程：
```java
//: concurrency/LiftOff.java
// Demonstration of the Runnable interface.

public class LiftOff implements Runnable {
  protected int countDown = 10; // Default
  private static int taskCount = 0;
  private final int id = taskCount++;
  public LiftOff() {}
  public LiftOff(int countDown) {
    this.countDown = countDown;
  }
  public String status() {
    return "#" + id + "(" +
      (countDown > 0 ? countDown : "Liftoff!") + "), ";
  }
  public void run() {
    while(countDown-- > 0) {
      System.out.print(status());
      Thread.yield();
    }
  }
} ///:~
```

Thread.yield() 的调用时对线程调度器的一种建议，它在声明：”我已经执行完生命周期中最重要的部分了，此刻正是切换给其他任务执行一段时间的大好时机“

```java
//: concurrency/MainThread.java

public class MainThread {
  public static void main(String[] args) {
    LiftOff launch = new LiftOff();
    launch.run();
  }
} /* Output:
#0(9), #0(8), #0(7), #0(6), #0(5), #0(4), #0(3), #0(2), #0(1), #0(Liftoff!),
*///:~
```

当从Runable导出一个类时，它必须具有run()方法，但是这个方法并无特殊之处–它不会产生任何内在的线程能力，要实现线程行为，你必须显示的调用一个任务附着到线程上

# Thread类
Thread构造器只需要一个Runable对象，调用Thread对象的start()方法为该线程执行必须的初始化操作。

例程：
```java
//: concurrency/BasicThreads.java
// The most basic use of the Thread class.

public class BasicThreads {
  public static void main(String[] args) {
    Thread t = new Thread(new LiftOff());
    t.start();
    System.out.println("Waiting for LiftOff");
  }
} /* Output: (90% match)
Waiting for LiftOff
#0(9), #0(8), #0(7), #0(6), #0(5), #0(4), #0(3), #0(2), #0(1), #0(Liftoff!),
*///:~
```
注意：main()和LiftOff.run()是程序中与其他线程“同时”执行的代码。

使用Thread与使用普通实现Runable接口对象的区别？

在使用普通对象时，这对于垃圾回收来说是一场公平的游戏，但是使用Thread时，情况就不同了，每个Thread都“注册”了它自己，因此确实有一个对它的引用。而且在它的任务退出其run()并死亡之前，垃圾回收期无法清除它。因此，一个线程会创建一个单独的执行线程，在对start()的调用完成之后。它仍然会继续存在

# 使用Executor
在Java SE5的Java.util.concurrent包中的执行器（Executor）将为你管理Thread对象，Executor在客户端和任务执行之间提供了一个间接层 
Executor在Java SE 5/6 中是启动任务的优选方法。

例程：
```java
//: concurrency/CachedThreadPool.java
import java.util.concurrent.*;

public class CachedThreadPool {
  public static void main(String[] args) {
    ExecutorService exec = Executors.newCachedThreadPool();
    for(int i = 0; i < 5; i++)
      exec.execute(new LiftOff());
    exec.shutdown();
  }
} /* Output: (Sample)
#0(9), #0(8), #1(9), #2(9), #3(9), #4(9), #0(7), #1(8), #2(8), #3(8), #4(8), #0(6), #1(7), #2(7), #3(7), #4(7), #0(5), #1(6), #2(6), #3(6), #4(6), #0(4), #1(5), #2(5), #3(5), #4(5), #0(3), #1(4), #2(4), #3(4), #4(4), #0(2), #1(3), #2(3), #3(3), #4(3), #0(1), #1(2), #2(2), #3(2), #4(2), #0(Liftoff!), #1(1), #2(1), #3(1), #4(1), #1(Liftoff!), #2(Liftoff!), #3(Liftoff!), #4(Liftoff!),
*///:~
```

注意： 
ExecutorService对象是使用静态的Executor方法创建的，这个方法可以确定Executor的类型，类型有CachedThreadPool，FixedThreadPool， 
SingleThreadPool。

shutdown()方法的调用可以防止新任务被提交给这个Executor，当前线程将继续执行在shutdown()被调用之前提交的所以任务，这个程序将在Executor中的所有任务完成之后尽快退出。

FixedThreadPool使用了有限的线程集来执行所提交的任务。 
例程：
```java
//: concurrency/FixedThreadPool.java
import java.util.concurrent.*;

public class FixedThreadPool {
  public static void main(String[] args) {
    // Constructor argument is number of threads:
    ExecutorService exec = Executors.newFixedThreadPool(5);
    for(int i = 0; i < 5; i++)
      exec.execute(new LiftOff());
    exec.shutdown();
  }
} /* Output: (Sample)
#0(9), #0(8), #1(9), #2(9), #3(9), #4(9), #0(7), #1(8), #2(8), #3(8), #4(8), #0(6), #1(7), #2(7), #3(7), #4(7), #0(5), #1(6), #2(6), #3(6), #4(6), #0(4), #1(5), #2(5), #3(5), #4(5), #0(3), #1(4), #2(4), #3(4), #4(4), #0(2), #1(3), #2(3), #3(3), #4(3), #0(1), #1(2), #2(2), #3(2), #4(2), #0(Liftoff!), #1(1), #2(1), #3(1), #4(1), #1(Liftoff!), #2(Liftoff!), #3(Liftoff!), #4(Liftoff!),
*///:~
```

有了FixedThreadPool在事件驱动的系统中，需要线程的事件处理器，通过直接从池中获取线程，也可以如你所愿地尽快的得到服务。你不会滥用可获得的资源。

SingleThreadPool就像是数量为1的FixedThreadPool 
向SingleThreadPool提交多个任务，那么这些任务将排队，每个任务都会在下一个任务开始之前结束。所有的任务将使用相同的线程。

# 从任务产生返回值
Runnable不返回任何值，如果希望任务在完成时能够返回一个值。那么可以实现Callable接口,在Java SE5中引入的Callable是一种具有参数类型的泛型，它从方法call()中返回值，必须使用ExecutorService.submit()方法调用它。

例程：
```java
//: concurrency/CallableDemo.java
import java.util.concurrent.*;
import java.util.*;

class TaskWithResult implements Callable<String> {
  private int id;
  public TaskWithResult(int id) {
    this.id = id;
  }
  public String call() {
    return "result of TaskWithResult " + id;
  }
}

public class CallableDemo {
  public static void main(String[] args) {
    ExecutorService exec = Executors.newCachedThreadPool();
    ArrayList<Future<String>> results =
      new ArrayList<Future<String>>();
    for(int i = 0; i < 10; i++)
      results.add(exec.submit(new TaskWithResult(i)));
    for(Future<String> fs : results)
      try {
        // get() blocks until completion:
        System.out.println(fs.get());
      } catch(InterruptedException e) {
        System.out.println(e);
        return;
      } catch(ExecutionException e) {
        System.out.println(e);
      } finally {
        exec.shutdown();
      }
  }
} /* Output:
result of TaskWithResult 0
result of TaskWithResult 1
result of TaskWithResult 2
result of TaskWithResult 3
result of TaskWithResult 4
result of TaskWithResult 5
result of TaskWithResult 6
result of TaskWithResult 7
result of TaskWithResult 8
result of TaskWithResult 9
*///:~
```

（注：内容整理自《Thinking in Java》）
