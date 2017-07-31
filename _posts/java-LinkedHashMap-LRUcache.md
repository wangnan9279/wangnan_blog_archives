---
title: Java-LinkedHashMap与LRUcache整理
link_title: java-LinkedHashMap-LRUcache
date: 2017-07-19 15:58:09
tags: [Java, JavaCollection]
categories: Java
thumbnailImage: http://onxkn9cbz.bkt.clouddn.com/28.jpg	
thumbnailImagePosition: left
---
<span/>
<!-- more -->
![](http://onxkn9cbz.bkt.clouddn.com/28.jpg)
<!-- toc -->
# LRU 缓存介绍
我们平时总会有一个电话本记录所有朋友的电话，但是，如果有朋友经常联系，那些朋友的电话号码不用翻电话本我们也能记住，但是，如果长时间没有联系了，要再次联系那位朋友的时候，我们又不得不求助电话本，但是，通过电话本查找还是很费时间的。但是，我们大脑能够记住的东西是一定的，**我们只能记住自己最熟悉的，而长时间不熟悉的自然就忘记了**。

其实，计算机也用到了同样的一个概念，我们用缓存来存放以前读取的数据，而不是直接丢掉，这样，再次读取的时候，可以直接在缓存里面取，而不用再重新查找一遍，这样系统的反应能力会有很大提高。但是，**当我们读取的个数特别大的时候，我们不可能把所有已经读取的数据都放在缓存里，毕竟内存大小是一定的，我们一般把最近常读取的放在缓存里（相当于我们把最近联系的朋友的姓名和电话放在大脑里一样）**。

LRU 缓存利用了这样的一种思想。**LRU 是 Least Recently Used 的缩写**，翻译过来就是“最近最少使用”，也就是说，**LRU 缓存把最近最少使用的数据移除，让给最新读取的数据。而往往最常读取的，也是读取次数最多的，所以，利用 LRU 缓存，我们能够提高系统的 performance。**

# 实现
使用LinkedHashMap

好处：
- **它本身已经实现了按照访问顺序的存储**，也就是说，最近读取的会放在最前面，最最不常读取的会放在最后（当然，它也可以实现按照插入顺序存储）。
- LinkedHashMap 本身有一个方法用于判断是否需要移除最不常读取的数，但是，原始方法默认不需要移除，所以，**我们需要 override 这样一个方法，使得当缓存里存放的数据个数超过规定个数后，就把最不常用的移除掉。**

# 代码
```java
public class LRUCache {
    private int capacity;
    private Map<Integer, Integer> cache;
    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.cache = new java.util.LinkedHashMap<Integer, Integer> (capacity, 0.75f, true) {
            // 定义put后的移除规则，大于容量就删除eldest
            protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
                return size() > capacity;
            }
        };
    }
    public int get(int key) {
        if (cache.containsKey(key)) {
            return cache.get(key);
        } else
            return -1;
    }
    public void set(int key, int value) {
        cache.put(key, value);
    }
}
```

参考：
- http://wiki.jikexueyuan.com/project/java-collection/linkedhashmap-lrucache.html
- https://yikun.github.io/2015/04/03/%E5%A6%82%E4%BD%95%E8%AE%BE%E8%AE%A1%E5%AE%9E%E7%8E%B0%E4%B8%80%E4%B8%AALRU-Cache%EF%BC%9F/

