---
title: Java ThreadLocal 简介
date: 2016-06-23 16:44:22
tags: [Java, JavaConcurrent]
categories: Java
link_title: java-ThreadLocal
thumbnailImage: https://upload-images.jianshu.io/upload_images/79431-ecfaca092a4a12a0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/300/format/webp
thumbnailImagePosition: left
---
<!-- toc -->
<!-- more -->
![](https://upload-images.jianshu.io/upload_images/79431-ecfaca092a4a12a0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/300/format/webp)
> ThreadLocal在Spring中发挥着重要的作用，在管理request作用域的Bean、事务管理、任务调度、AOP等模块都出现了它们的身影，起着举足轻重的作用。要想了解Spring事务管理的底层技术，ThreadLocal是必须攻克的山头堡垒。


我们知道spring通过各种模板类降低了开发者使用各种数据持久技术的难度。这些模板类都是线程安全的，也就是说，多个DAO可以复用同一个模板实例而不会发生冲突。我们使用模板类访问底层数据，根据持久化技术的不同，模板类需要绑定数据连接或会话的资源。但这些资源本身是非线程安全的，也就是说它们不能在同一时刻被多个线程共享。虽然模板类通过资源池获取数据连接或会话，但资源池本身解决的是数据连接或会话的缓存问题，并非数据连接或会话的线程安全问题。

按照传统经验，如果某个对象是非线程安全的，在多线程环境下，对对象的访问必须采用synchronized进行线程同步。但模板类并未采用线程同步机制，因为线程同步会降低并发性，影响系统性能。此外，通过代码同步解决线程安全的挑战性很大，可能会增强好几倍的实现难度。那么模板类究竟仰仗何种魔法神功，可以在无须线程同步的情况下就化解线程安全的难题呢？答案就是ThreadLocal！

ThreadLocal在Spring中发挥着重要的作用，在管理request作用域的Bean、事务管理、任务调度、AOP等模块都出现了它们的身影，起着举足轻重的作用。要想了解Spring事务管理的底层技术，ThreadLocal是必须攻克的山头堡垒。

# ThreadLocal是什么
早在JDK 1.2的版本中就提供Java.lang.ThreadLocal，ThreadLocal为解决多线程程序的并发问题提供了一种新的思路。使用这个工具类可以很简洁地编写出优美的多线程程序。 
ThreadLocal，顾名思义，它不是一个线程，而是线程的一个本地化对象。当工作于多线程中的对象使用ThreadLocal维护变量时，ThreadLocal为每个使用该变量的线程分配一个独立的变量副本。所以每一个线程都可以独立地改变自己的副本，而不会影响其他线程所对应的副本。从线程的角度看，这个变量就像是线程的本地变量，这也是类名中“Local”所要表达的意思。

线程局部变量并不是Java的新发明，很多语言（如IBM XL、FORTRAN）在语法层面就提供线程局部变量。在Java中没有提供语言级支持，而以一种变通的方法，通过ThreadLocal的类提供支持。所以，在Java中编写线程局部变量的代码相对来说要笨拙一些，这也是为什么线程局部变量没有在Java开发者中得到很好普及的原因。

# ThreadLocal的接口方法
ThreadLocal类接口很简单，只有4个方法，我们先来了解一下。 
void set(Object value) 
设置当前线程的线程局部变量的值； 
public Object get() 
该方法返回当前线程所对应的线程局部变量； 
public void remove() 
将当前线程局部变量的值删除，目的是为了减少内存的占用，该方法是JDK 5.0新增的方法。需要指出的是，当线程结束后，对应该线程的局部变量将自动被垃圾回收，所以显式调用该方法清除线程的局部变量并不是必须的操作，但它可以加快内存回收的速度； 
protected Object initialValue() 
返回该线程局部变量的初始值，该方法是一个protected的方法，显然是为了让子类覆盖而设计的。这个方法是一个延迟调用方法，在线程第1次调用get()或set(Object)时才执行，并且仅执行1次。ThreadLocal中的默认实现直接返回一个null。

值得一提的是，在JDK5.0中，ThreadLocal已经支持泛型，该类的类名已经变为ThreadLocal。API方法也相应进行了调整，新版本的API方法分别是void set(T value)、T get()以及T initialValue()。

ThreadLocal是如何做到为每一个线程维护变量的副本的呢？其实实现的思路很简单：在ThreadLocal类中有一个Map，用于存储每一个线程的变量副本，Map中元素的键为线程对象，而值对应线程的变量副本。我们自己就可以提供一个简单的实现版本：

代码清单9-3 SimpleThreadLocal
```java
public class SimpleThreadLocal {  
    private Map valueMap = Collections.synchronizedMap(new HashMap());  
    public void set(Object newValue) {  
                //①键为线程对象，值为本线程的变量副本  
        valueMap.put(Thread.currentThread(), newValue);  
    }  
    public Object get() {  
        Thread currentThread = Thread.currentThread();  

                //②返回本线程对应的变量  
        Object o = valueMap.get(currentThread);   

                //③如果在Map中不存在，放到Map中保存起来  
               if (o == null && !valueMap.containsKey(currentThread)) {  
            o = initialValue();  
            valueMap.put(currentThread, o);  
        }  
        return o;  
    }  
    public void remove() {  
        valueMap.remove(Thread.currentThread());  
    }  
    public Object initialValue() {  
        return null;  
    }  
}  
```
虽然代码清单9 3中这个ThreadLocal实现版本显得比较幼稚，但它和JDK所提供的ThreadLocal类在实现思路上是非常相近的。

一个TheadLocal实例
下面，我们通过一个具体的实例了解一下ThreadLocal的具体使用方法。
代码清单9-4 SequenceNumber

```java
package com.baobaotao.basic;  

public class SequenceNumber {  

        //①通过匿名内部类覆盖ThreadLocal的initialValue()方法，指定初始值  
    private static ThreadLocal<Integer> seqNum = new ThreadLocal<Integer>(){  
        public Integer initialValue(){  
            return 0;  
        }  
    };  

        //②获取下一个序列值  
    public int getNextNum(){  
        seqNum.set(seqNum.get()+1);  
        return seqNum.get();  
    }  

    public static void main(String[ ] args)   
    {  
          SequenceNumber sn = new SequenceNumber();  

         //③ 3个线程共享sn，各自产生序列号  
         TestClient t1 = new TestClient(sn);    
         TestClient t2 = new TestClient(sn);  
         TestClient t3 = new TestClient(sn);  
         t1.start();  
         t2.start();  
         t3.start();  
    }     
    private static class TestClient extends Thread  
    {  
        private SequenceNumber sn;  
        public TestClient(SequenceNumber sn) {  
            this.sn = sn;  
        }  
        public void run()  
        {  
                        //④每个线程打出3个序列值  
            for (int i = 0; i < 3; i++) {  
            System.out.println("thread["+Thread.currentThread().getName()+  
"] sn["+sn.getNextNum()+"]");  
            }  
        }  
    }  
}  
```

通常我们通过匿名内部类的方式定义ThreadLocal的子类，提供初始的变量值，如①处所示。TestClient线程产生一组序列号，在③处，我们生成3个TestClient，它们共享同一个SequenceNumber实例。运行以上代码，在控制台上输出以下的结果：

> thread[Thread-2] sn[1] 
thread[Thread-0] sn[1] 
thread[Thread-1] sn[1] 
thread[Thread-2] sn[2] 
thread[Thread-0] sn[2] 
thread[Thread-1] sn[2] 
thread[Thread-2] sn[3] 
thread[Thread-0] sn[3] 
thread[Thread-1] sn[3]

考查输出的结果信息，我们发现每个线程所产生的序号虽然都共享同一个Sequence Number实例，但它们并没有发生相互干扰的情况，而是各自产生独立的序列号，这是因为我们通过ThreadLocal为每一个线程提供了单独的副本。

# 与Thread同步机制的比较
ThreadLocal和线程同步机制相比有什么优势呢？ThreadLocal和线程同步机制都是为了解决多线程中相同变量的访问冲突问题。

在同步机制中，通过对象的锁机制保证同一时间只有一个线程访问变量。这时该变量是多个线程共享的，使用同步机制要求程序缜密地分析什么时候对变量进行读写，什么时候需要锁定某个对象，什么时候释放对象锁等繁杂的问题，程序设计和编写难度相对较大。

而ThreadLocal则从另一个角度来解决多线程的并发访问。ThreadLocal为每一个线程提供一个独立的变量副本，从而隔离了多个线程对访问数据的冲突。因为每一个线程都拥有自己的变量副本，从而也就没有必要对该变量进行同步了。ThreadLocal提供了线程安全的对象封装，在编写多线程代码时，可以把不安全的变量封装进ThreadLocal。

由于ThreadLocal中可以持有任何类型的对象，低版本JDK所提供的get()返回的是Object对象，需要强制类型转换。但JDK 5.0通过泛型很好的解决了这个问题，在一定程度上简化ThreadLocal的使用，代码清单9-2就使用了JDK 5.0新的ThreadLocal版本。

概括起来说，对于多线程资源共享的问题，同步机制采用了“以时间换空间”的方式：访问串行化，对象共享化。而ThreadLocal采用了“以空间换时间”的方式：访问并行化，对象独享化。前者仅提供一份变量，让不同的线程排队访问，而后者为每一个线程都提供了一份变量，因此可以同时访问而互不影响。
# Spring使用ThreadLocal解决线程安全问题
我们知道在一般情况下，只有无状态的Bean才可以在多线程环境下共享，在Spring中，绝大部分Bean都可以声明为singleton作用域。就是因为Spring对一些Bean（如RequestContextHolder、TransactionSynchronizationManager、LocaleContextHolder等）中非线程安全的“状态性对象”采用ThreadLocal进行封装，让它们也成为线程安全的“状态性对象”，因此有状态的Bean就能够以singleton的方式在多线程中正常工作了。

一般的Web应用划分为展现层、服务层和持久层三个层次，在不同的层中编写对应的逻辑，下层通过接口向上层开放功能调用。在一般情况下，从接收请求到返回响应所经过的所有程序调用都同属于一个线程，如图9-2所示。

![](java-ThreadLocal/01.png)

这样用户就可以根据需要，将一些非线程安全的变量以ThreadLocal存放，在同一次请求响应的调用线程中，所有对象所访问的同一ThreadLocal变量都是当前线程所绑定的。 
下面的实例能够体现Spring对有状态Bean的改造思路：

代码清单9-5 TopicDao：非线程安全
```java
public class TopicDao {  
   //①一个非线程安全的变量  
   private Connection conn;   
   public void addTopic(){  
        //②引用非线程安全变量  
       Statement stat = conn.createStatement();  
       …  
   }  
}  
```
由于①处的conn是成员变量，因为addTopic()方法是非线程安全的，必须在使用时创建一个新TopicDao实例（非singleton）。下面使用ThreadLocal对conn这个非线程安全的“状态”进行改造：

代码清单9-6 TopicDao：线程安全
```java
import java.sql.Connection;  
import java.sql.Statement;  
public class TopicDao {  

  //①使用ThreadLocal保存Connection变量  
private static ThreadLocal<Connection> connThreadLocal = new ThreadLocal<Connection>();  
public static Connection getConnection(){  

        //②如果connThreadLocal没有本线程对应的Connection创建一个新的Connection，  
        //并将其保存到线程本地变量中。  
if (connThreadLocal.get() == null) {  
            Connection conn = ConnectionManager.getConnection();  
            connThreadLocal.set(conn);  
              return conn;  
        }else{  
              //③直接返回线程本地变量  
            return connThreadLocal.get();  
        }  
    }  
    public void addTopic() {  

        //④从ThreadLocal中获取线程对应的  
         Statement stat = getConnection().createStatement();  
    }  
}  
```
不同的线程在使用TopicDao时，先判断connThreadLocal.get()是否为null，如果为null，则说明当前线程还没有对应的Connection对象，这时创建一个Connection对象并添加到本地线程变量中；如果不为null，则说明当前的线程已经拥有了Connection对象，直接使用就可以了。这样，就保证了不同的线程使用线程相关的Connection，而不会使用其他线程的Connection。因此，这个TopicDao就可以做到singleton共享了。

当然，这个例子本身很粗糙，将Connection的ThreadLocal直接放在Dao只能做到本Dao的多个方法共享Connection时不发生线程安全问题，但无法和其他Dao共用同一个Connection，要做到同一事务多Dao共享同一个Connection，必须在一个共同的外部类使用ThreadLocal保存Connection。但这个实例基本上说明了Spring对有状态类线程安全化的解决思路。在本章后面的内容中，我们将详细说明Spring如何通过ThreadLocal解决事务管理的问题。