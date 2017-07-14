---
title: Java-ConcurrentHashMap整理
link_title: java-concurrentHashMap
date: 2017-06-15 16:20:42
tags: [Java,Map]
categories: Java
thumbnailImage: http://onxkn9cbz.bkt.clouddn.com/concurrentHashMap.png
thumbnailImagePosition: left
---
<!-- toc -->
<!-- more -->
http://onxkn9cbz.bkt.clouddn.com/concurrentHashMap.png
参考：
- http://blog.csdn.net/qq_27093465/article/details/52279473
- http://blog.csdn.net/u010723709/article/details/48007881
- https://yq.aliyun.com/articles/36781

# 先说说HashMap HashTable
## HashMap
- 线程不安全，并发情况下不能使用
## HashTable:
- 线程安全，但是效率低下
- HashTable容器使用**synchronized**来保证线程安全，但在线程竞争激烈的情况下HashTable的效率非常低下。因为当一个线程访问HashTable的同步方法时，其他线程访问HashTable的同步方法时，可能会进入阻塞或轮询状态。
- 线程1使用put方法时，线程2不但不能使用put方法，也不能使用get方法

# ConcurrentHashMap(jdk1.8之前)
## 简介
- 假如容器里有多把锁，每一把锁用于锁容器其中一部分数据，那么当多线程访问容器里不同数据段的数据时，线程间就不会存在锁竞争，从而可以有效的提高并发访问效率，这就是ConcurrentHashMap所使用的**锁分段技术**，首先将数据分成一段一段的存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个段数据的时候，其他段的数据也能被其他线程访问。
- ConcurrentHashMap是由Segment数组结构和HashEntry数组结构组成。Segment是一种可重入锁**ReentrantLock**，HashEntry则用于存储键值对数据。一个ConcurrentHashMap里包含多个Segment数组，Segment的结构和HashMap类似，是一种数组和链表结构，一个Segment里包含多个HashEntry数组，每个HashEntry是一个链表结构的元素，,当对HashEntry数组的数据进行修改时，必须首先获得它对应的Segment锁。
![类图](concurrentHashMap-note/01.png)
![segment](concurrentHashMap-note/02.png)

# ConcurrentHashMap(jdk1.8)
## 简介
- 与JDK6的版本有很大的差异。实现线程安全的思想也已经完全变了，它摒弃了Segment（锁段）的概念，而是启用了一种全新的方式实现,利用**CAS算法**。它沿用了与它同时期的HashMap版本的思想，底层依然由“数组”+链表+红黑树的方式思想，但是为了做到并发，又增加了很多辅助的类，例如TreeBin，Traverser等对象内部类。

## 重要属性
### sizeCtl
可以说它是ConcurrentHashMap中出镜率很高的一个属性，因为它是一个控制标识符，在不同的地方有不同用途，而且它的取值不同，也代表不同的含义。
- 负数代表正在进行初始化或扩容操作
- -1代表正在初始化
- -N 表示有N-1个线程正在进行扩容操作
- 正数或0代表hash表还没有被初始化，这个数值表示初始化或下一次进行扩容的大小，这一点类似于扩容阈值的概念。还后面可以看到，它的值始终是当前ConcurrentHashMap容量的0.75倍，这与loadfactor是对应的。实际容量>=sizeCtl，则扩容

### concurrencyLevel
concurrencyLevel，能够同时更新ConccurentHashMap且不产生锁竞争的最大线程数，在Java8之前实际上就是ConcurrentHashMap中的分段锁个数，即Segment[]的数组长度。正确地估计很重要，当低估，数据结构将根据额外的竞争，从而导致线程试图写入当前锁定的段时阻塞；相反，如果高估了并发级别，你遇到过大的膨胀，由于段的不必要的数量; 这种膨胀可能会导致性能下降，由于高数缓存未命中。在Java8里，仅仅是为了兼容旧版本而保留。唯一的作用就是保证构造map时初始容量不小于concurrencyLevel。

## 重要内部类
### Node
Node是最核心的内部类，它包装了key-value键值对，所有插入ConcurrentHashMap的数据都包装在这里面。它与HashMap中的定义很相似，**但是有一些差别它对value和next属性设置了volatile同步锁，它不允许调用setValue方法直接改变Node的value域，它增加了find方法辅助map.get()方法。**
## TreeNode
- 链表>8，才可能转为TreeNode.
- HashMap的TreeNode继承至LinkedHashMap.Entry；而这里继承至自己实现的Node，将带有next指针，便于treebin访问。
- 树节点类，另外一个核心的数据结构。当链表长度过长的时候，会转换为TreeNode。但是与HashMap不相同的是，**它并不是直接转换为红黑树，而是把这些结点包装成TreeNode放在TreeBin对象中，由TreeBin完成对红黑树的包装**。而且TreeNode在ConcurrentHashMap集成自Node类，而并非HashMap中的集成自LinkedHashMap.Entry<K,V>类，也就是说TreeNode带有next指针，这样做的目的是方便基于TreeBin的访问。
## TreeBin
- TreeBin用于封装维护TreeNode，包含putTreeVal、lookRoot、UNlookRoot、remove、balanceInsetion、balanceDeletion等方法
- 这个类并不负责包装用户的key、value信息，而是包装的很多TreeNode节点。它代替了TreeNode的根节点**，也就是说在实际的ConcurrentHashMap“数组”中，存放的是TreeBin对象，而不是TreeNode对象**，这是与HashMap的区别。另外这个类还带有了读写锁。

##  Unsafe与CAS
在ConcurrentHashMap中，随处可以看到U, 大量使用了U.compareAndSwapXXX的方法，这个方法是利用一个CAS算法实现无锁化的修改值的操作，他可以大大降低锁代理的性能消耗。**这个算法的基本思想就是不断地去比较当前内存中的变量值与你指定的一个变量值是否相等，如果相等，则接受你指定的修改的值，否则拒绝你的操作**。因为当前线程中的值已经不是最新的值，你的修改很可能会覆盖掉其他线程修改的结果。这一点与乐观锁，SVN的思想是比较类似的。

##  3个原子操作
1. tabAt // 获取索引i处Node
2. casTabAt // 利用CAS算法设置i位置上的Node节点（将c和table[i]比较，相同则插入v）。
3. setTabAt // 设置节点位置的值，仅在上锁区被调用

## put相关
这个put方法依然沿用HashMap的put方法的思想，根据hash值计算这个新插入的点在table中的位置i，**如果i位置是空的，直接放进去，否则进行判断，如果i位置是树节点，按照树的方式插入新的节点，否则把i插入到链表的末尾。**ConcurrentHashMap中依然沿用这个思想，**有一个最重要的不同点就是ConcurrentHashMap不允许key或value为null值**。另外由于涉及到多线程，put方法就要复杂一点。在多线程中可能有以下两个情况
- 如果一个或多个线程正在对ConcurrentHashMap进行扩容操作，当前线程也要进入扩容的操作中。这个扩容的操作之所以能被检测到，是因为transfer方法中在空结点上插入forward节点，如果检测到需要插入的位置被forward节点占有，就帮助进行扩容
- 如果检测到要插入的节点是非空且不是forward节点，就对这个节点加锁，这样就保证了线程安全。尽管这个有一些影响效率，但是还是会比hashTable的synchronized要好得多

流程：
1. 判空：null直接抛空指针异常
2. hash：计算哈希值
3. 遍历table
- 若table为空，则初始化，仅设置相关参数；
- 计算当前key存放位置，即table的下标i=(n - 1) & hash；
- 若待存放位置为null，casTabAt无锁插入；
- 若是forwarding nodes（检测到正在扩容），则helpTransfer（帮助其扩容）；
- else（待插入位置非空且不是forward节点，即碰撞了），将头节点上锁（保证了线程安全）：区分链表节点和树节点，分别插入（遇到hash值与key值都与新节点一致的情况，只需要更新value值即可。否则依次向后遍历，直到链表尾插入这个结点）；
- 若链表长度>8，则treeifyBin转树（Note：若length<64,直接tryPresize,两倍table.length;不转树）。
4. addCount

## resize相关 
当ConcurrentHashMap容量不足的时候，需要对table进行扩容。这个方法的基本思想跟HashMap是很像的，但是由于它是支持并发扩容的，所以要复杂的多。原因是它支持多线程进行扩容操作，而并没有加锁。我想这样做的目的不仅仅是为了满足concurrent的要求，而是希望利用并发处理去减少扩容带来的时间影响**。因为在扩容的时候，总是会涉及到从一个“数组”到另一个“数组”拷贝的操作**，如果这个操作能够并发进行，那真真是极好的了。
整个扩容操作分为两个部分:
- 第一部分是构建一个nextTable,它的容量是原来的两倍，这个操作是单线程完成的。这个单线程的保证是通过RESIZE_STAMP_SHIFT这个常量经过一次运算来保证的，这个地方在后面会有提到；
- 第二个部分就是将原来table中的元素复制到nextTable中，这里允许多线程进行操作。

多线程又是如何实现的呢？
 遍历到ForwardingNode节点((fh = f.hash) == MOVED)，说明此节点被处理过了，直接跳过。这是控制并发扩容的核心 。由于给节点上了锁，只允许当前线程完成此节点的操作，处理完毕后，将对应值设为ForwardingNode（fwd），其他线程看到forward，直接向后遍历。如此便完成了多线程的复制工作，也解决了线程安全问题。


# Size相关
由于ConcurrentHashMap在统计size时可能正被多个线程操作，而我们又不可能让他停下来让我们计算，所以只能计量一个**估计值**。