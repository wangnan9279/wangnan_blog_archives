---
title: Java-ArrayList快速失败机制/CopyOnWriteArrayList/扩容
link_title: java-arrayList
date: 2017-07-31 14:51:55
tags: [Java, JavaCollection]
categories: Java
thumbnailImage: http://onxkn9cbz.bkt.clouddn.com/33.jpg	
thumbnailImagePosition: left
---
<span/>
<!-- more -->
![](http://onxkn9cbz.bkt.clouddn.com/33.jpg)
<!-- toc -->

# 1.迭代ArrayList时做add或remove操作会发生什么？
## 答案
会抛出 java.util.ConcurrentModificationException 

## 解决方法
1. 对JAVA集合进行遍历删除时务必要用迭代器
2. 使用CopyOnWriteArrayList

## 总结：
1. **对于ArrayList，在使用Iterator遍历时，不能使用list.add()、list.remove()等改变list的操作，只能用it.remove()**
原因是ArrayList不是线程安全的，需在单线程环境下使用，如果在遍历时还有别的线程做增删操作，必然会有问题，如数组下标越界
ArrayList#Iterator设计的是不能在迭代时有别的线程对list修改，此种修改对当前迭代器是可能存在问题的，所以增加了对modCount的校验
但当前迭代器可以remove，因为它自己删除就不是并发修改了
迭代器remove会重置expectedModCount，并将cursor往前一位
2. CopyOnWriteArrayList在使用Iterator遍历时，**可以用list.add()，list.remove()等改变list的操作，但不支持it.remove()**
首先CopyOnWriteArrayList是用于多线程环境下的
因为CopyOnWriteArrayList的Iterator实现类COWIterator会在创建时复制一份list的副本，之后迭代的是副本
**所以期间怎么对list.add()，list.remove()都没事，list.remove()操作的是原始的list
但不支持it.remove()**
因为Iterator中的本身就是副本，删除副本中的元素没意义，如果去删除原始list，在并发环境下此时list可能和创建迭代器时的副本已经完全不同了

## 快速失败（fast-fail）机制
**快速失败"即fail-fast,它是java集合的一种错误检测机制。当多钱程对集合进行结构上的改变或者集合在迭代元素时直接调用自身方法改变集合结构而没有通知迭代器时，有可能会触发fast-fail机制并抛出异常**。需要注意一点，有可能触发fast-fail机制而不是肯定。触发时机是在迭代过程中，集合的结构发生了变化而此时迭代器并不知道或者还没来得及反应时便会产生fail-fast机制。这里再次强调，迭代器的快速失败行为无法得到保证，因为一般来说，不可能对是否出现不同步并发修改或者自身修改做出任何硬性保证。快速失败迭代器会尽最大努力抛出 ConcurrentModificationException。

# 2.CopyOnWriteArrayList
## 概述
Copy-On-Write简称COW，是一种用于程序设计中的优化策略。其基本思路是，**从一开始大家都在共享同一个内容，当某个人想要修改这个内容的时候，才会真正把内容Copy出去形成一个新的内容然后再改，这是一种延时懒惰策略。从JDK1.5开始Java并发包里提供了两个使用CopyOnWrite机制实现的并发容器,它们是CopyOnWriteArrayList和CopyOnWriteArraySet。**CopyOnWrite容器非常有用，可以在非常多的并发场景中使用到。

## 什么是CopyOnWrite容器
CopyOnWrite容器即写时复制的容器。**通俗的理解是当我们往一个容器添加元素的时候，不直接往当前容器添加，而是先将当前容器进行Copy，复制出一个新的容器，然后新的容器里添加元素，添加完元素之后，再将原容器的引用指向新的容器。这样做的好处是我们可以对CopyOnWrite容器进行并发的读，而不需要加锁，因为当前容器不会添加任何元素。所以CopyOnWrite容器也是一种读写分离的思想，读和写不同的容器。**

## 实现原理
在使用CopyOnWriteArrayList之前，我们先阅读其源码了解下它是如何实现的。以下代码是向CopyOnWriteArrayList中add方法的实现（向CopyOnWriteArrayList里添加元素），可以发现在添加的时候是需要加锁的，否则多线程写的时候会Copy出N个副本出来。
```java
/**
     * Appends the specified element to the end of this list.
     *
     * @param e element to be appended to this list
     * @return <tt>true</tt> (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
    }
```
　读的时候不需要加锁，如果读的时候有多个线程正在向CopyOnWriteArrayList添加数据，读还是会读到旧的数据，因为写的时候不会锁住旧的CopyOnWriteArrayList。
```java
public E get(int index) {
    return get(getArray(), index);
}
```
JDK中并没有提供CopyOnWriteMap，我们可以参考CopyOnWriteArrayList来实现一个，基本代码如下：
```java
import java.util.Collection;
import java.util.Map;
import java.util.Set;
 
public class CopyOnWriteMap<K, V> implements Map<K, V>, Cloneable {
    private volatile Map<K, V> internalMap;
 
    public CopyOnWriteMap() {
        internalMap = new HashMap<K, V>();
    }
 
    public V put(K key, V value) {
 
        synchronized (this) {
            Map<K, V> newMap = new HashMap<K, V>(internalMap);
            V val = newMap.put(key, value);
            internalMap = newMap;
            return val;
        }
    }
 
    public V get(Object key) {
        return internalMap.get(key);
    }
 
    public void putAll(Map<? extends K, ? extends V> newData) {
        synchronized (this) {
            Map<K, V> newMap = new HashMap<K, V>(internalMap);
            newMap.putAll(newData);
            internalMap = newMap;
        }
    }
}
```
实现很简单，只要了解了CopyOnWrite机制，我们可以实现各种CopyOnWrite容器，并且在不同的应用场景中使用。

## CopyOnWrite的应用场景
**CopyOnWrite并发容器用于读多写少的并发场景。比如白名单，黑名单**，商品类目的访问和更新场景，假如我们有一个搜索网站，用户在这个网站的搜索框中，输入关键字搜索内容，但是某些关键字不允许被搜索。这些不能被搜索的关键字会被放在一个黑名单当中，黑名单每天晚上更新一次。当用户搜索时，会检查当前关键字在不在黑名单当中，如果在，则提示不能搜索。实现
```java
package com.ifeve.book;
 
import java.util.Map;
 
import com.ifeve.book.forkjoin.CopyOnWriteMap;
 
/**
 * 黑名单服务
 *
 * @author fangtengfei
 *
 */
public class BlackListServiceImpl {
 
    private static CopyOnWriteMap<String, Boolean> blackListMap = new CopyOnWriteMap<String, Boolean>(
            1000);
 
    public static boolean isBlackList(String id) {
        return blackListMap.get(id) == null ? false : true;
    }
 
    public static void addBlackList(String id) {
        blackListMap.put(id, Boolean.TRUE);
    }
 
    /**
     * 批量添加黑名单
     *
     * @param ids
     */
    public static void addBlackList(Map<String,Boolean> ids) {
        blackListMap.putAll(ids);
    }
 
}
```
　代码很简单，但是使用CopyOnWriteMap需要注意两件事情：

　　1. 减少扩容开销。根据实际需要，初始化CopyOnWriteMap的大小，避免写时CopyOnWriteMap扩容的开销。

　　2. 使用批量添加。因为每次添加，容器每次都会进行复制，所以减少添加次数，可以减少容器的复制次数。如使用上面代码里的addBlackList方法。

## CopyOnWrite的缺点
CopyOnWrite容器有很多优点，但是同时也存在两个问题，即内存占用问题和数据一致性问题。所以在开发的时候需要注意一下。
### 内存占用问题
因为CopyOnWrite的写时复制机制，所以在进行写操作的时候，内存里会同时驻扎两个对象的内存，旧的对象和新写入的对象（注意:在复制的时候只是复制容器里的引用，只是在写的时候会创建新对象添加到新容器里，而旧容器的对象还在使用，所以有两份对象内存）。如果这些对象占用的内存比较大，比如说200M左右，那么再写入100M数据进去，内存就会占用300M，那么这个时候很有可能造成频繁的Yong GC和Full GC。之前我们系统中使用了一个服务由于每晚使用CopyOnWrite机制更新大对象，造成了每晚15秒的Full GC，应用响应时间也随之变长。
　　针对内存占用问题，可以通过压缩容器中的元素的方法来减少大对象的内存消耗，比如，如果元素全是10进制的数字，可以考虑把它压缩成36进制或64进制。或者不使用CopyOnWrite容器，而使用其他的并发容器，如ConcurrentHashMap。
　　
### 数据一致性问题
CopyOnWrite容器只能保证数据的最终一致性，不能保证数据的实时一致性。所以如果你希望写入的的数据，马上能读到，请不要使用CopyOnWrite容器。

下面这篇文章验证了CopyOnWriteArrayList和同步容器的性能：
http://blog.csdn.net/wind5shy/article/details/5396887
下面这篇文章简单描述了CopyOnWriteArrayList的使用：
http://blog.csdn.net/imzoer/article/details/9751591

# 扩容
- 以数组实现。节约空间，但数组有容量限制。超出限制时会增加50%容量，用System.arraycopy()复制到新的数组，因此最好能给出数组大小的预估值。默认第一次插入元素时创建大小为10的数组。

# 参考
- http://www.cnblogs.com/caiyao/p/4963206.html
- http://www.qingpingshan.com/m/view.php?aid=219803
- http://blog.csdn.net/smcwwh/article/details/7036663
- http://www.cnblogs.com/dolphin0520/p/3938914.html
- http://yikun.github.io/2015/04/04/Java-ArrayList%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/
- http://www.cnblogs.com/jeffwongishandsome/archive/2012/11/06/2753054.html

