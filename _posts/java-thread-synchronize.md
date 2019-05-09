---
title: Java-几种线程同步的方法
link_title: java-thread-synchronize
date: 2017-07-24 14:48:38
tags: [Java, JavaConcurrent]
categories: Java
thumbnailImage: https://upload-images.jianshu.io/upload_images/79431-d50a70be5057649e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/510/format/webp	
thumbnailImagePosition: left
---
<span/>
<!-- more -->
![](https://upload-images.jianshu.io/upload_images/79431-d50a70be5057649e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/510/format/webp)
<!-- toc -->
# 同步方法
即**有synchronized关键字修饰的方法**。 由于java的每个对象都有一个内置锁，当用此关键字修饰方法时，内置锁会保护整个方法。在调用该方法前，需要获得内置锁，否则就处于阻塞状态。
synchronized关键字也可以修饰静态方法，此时如果调用该静态方法，将会锁住整个方法

# 同步代码块
即**有synchronized关键字修饰的语句块**。被该关键字修饰的语句块会自动被加上内置锁，从而实现同步
**注：同步是一种高开销的操作，因此应该尽量减少同步的内容。通常没有必要同步整个方法，使用synchronized代码块同步关键代码即可**。

# 使用特殊域变量(Volatile)实现线程同步
1. volatile关键字为域变量的访问提供了一种**免锁机制**
2. 使用volatile修饰域**相当于告诉虚拟机该域可能会被其他线程更新**
3. **因此每次使用该域就要重新计算，而不是使用寄存器中的值**
4. **volatile不会提供任何原子操作，它也不能用来修饰final类型的变量**

就是因为volatile不能保证原子操作导致的，因此volatile不能代替synchronized。此外volatile会组织编译器对代码优化，因此能不使用它就不适用它吧。它的原理是每次**要线程要访问volatile修饰的变量时都是从内存中读取，而不是存缓存当中读取，因此每个线程访问到的变量值都是一样的。这样就保证了同步。**

# 使用重入锁实现线程同步
在JavaSE5.0中新增了一个java.util.concurrent包来支持同步。**ReentrantLock类是可重入、互斥、实现了Lock接口的锁**， 它与使用synchronized方法和快具有相同的基本行为和语义，并且扩展了其能力。
ReenreantLock类的常用方法有：
- ReentrantLock() : 创建一个ReentrantLock实例
- lock() : 获得锁
- unlock() : 释放锁
如果synchronized关键字能满足用户的需求，就用synchronized，因为它能简化代码 。**如果需要更高级的功能，就用ReentrantLock类，此时要注意及时释放锁，否则会出现死锁，通常在finally代码释放锁**

# 使用局部变量实现线程同步
如果使用**ThreadLocal**管理变量，则每一个使用该变量的线程都获得该变量的副本，副本之间相互独立，这样每一个线程都可以随意修改自己的变量副本，而不会对其他线程产生影响。现在明白了吧，原来每个线程运行的都是一个副本，也就是说存钱和取钱是两个账户，知识名字相同而已。所以就会发生上面的效果。
ThreadLocal与同步机制
1. ThreadLocal与同步机制都是为了解决多线程中相同变量的访问冲突问题
2. 前者采用以”空间换时间”的方法，后者采用以”时间换空间”的方式

参考：
- http://www.codeceo.com/article/java-multi-thread-sync.html
