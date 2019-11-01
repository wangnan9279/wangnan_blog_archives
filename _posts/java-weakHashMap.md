---
layout: post
title: Java-WeakHashMap整理
date: 2017-07-07 16:27:01
tags: [Java, JavaCollection]
categories: Java
comments: true
thumbnailImage: https://i.loli.net/2019/09/24/OGSmnYX6PbdvyWZ.png
thumbnailImagePosition: left
link_title: WeakHashMap
---
<!-- toc -->
<!-- more -->
![](https://i.loli.net/2019/09/24/OGSmnYX6PbdvyWZ.png)
参考：
- http://mikewang.blog.51cto.com/3826268/880775
- http://www.wanghd.com/%E5%90%8E%E7%AB%AF/2012/08/01/weakhashmapzuo-yong-he-shi-li.html

# 介绍
1. 以弱键 实现的基于哈希表的 Map。在 WeakHashMap 中，当某个键不再正常使用时，将自动移除其条目。更精确地说，**对于一个给定的键，其映射的存在并不阻止垃圾回收器对该键的丢弃，这就使该键成为可终止的，被终止，然后被回收。丢弃某个键时，其条目从映射中有效地移除**
2. WeakHashMap 类的行为部分取决于垃圾回收器的动作。因为垃圾回收器在任何时候都可能丢弃键，WeakHashMap 就像是一个被悄悄移除条目的未知线程。特别地，即使对 WeakHashMap 实例进行同步，并且没有调用任何赋值方法，在一段时间后 size 方法也可能返回较小的值，对于 isEmpty 方法，返回 false，然后返回true，对于给定的键，containsKey 方法返回 true 然后返回 false，对于给定的键，get 方法返回一个值，但接着返回 null，对于以前出现在映射中的键，put 方法返回 null，而 remove 方法返回 false，对于键 set、值 collection 和条目 set 进行的检查，生成的元素数量越来越少。
3. **WeakHashMap 中的每个键对象间接地存储为一个弱引用的指示对象**。因此，不管是在映射内还是在映射之外，只有在垃圾回收器清除某个键的弱引用之后，该键才会自动移除。

# 作用
WeekHashMap 的这个特点特别适用于需要缓存的场景。在缓存场景下，由于内存是有限的，不能缓存所有对象；对象缓存命中可以提高系统效率，但缓存MISS也不会造成错误，因为可以通过计算重新得到。

# 代码解释
## 测试一下保持引用会不会被自动释放
```java
package wanghuida.test;

import java.util.WeakHashMap;
import java.util.Map;

import java.util.List;
import java.util.ArrayList;

public class Entry {

    /**
     * @param args
     */
    public static void main(String[] args) {
        // TODO Auto-generated method stub
        List<String[]> templist = new ArrayList<String[]>();  
        //设的多一点，可以让GC真实发挥
        for(int i=0; i < 1000000; i++){
            String[] tempstr = new String[2];
            templist.add(tempstr);
        }
        
        Map<String[], String[]> map = new WeakHashMap<String[], String[]>();
        for(int i=0; i < 100; i++){
            map.put(templist.get(i), new String[2]);
            System.gc();
            System.out.println(map.size());
        }

    }

}
```
输出1，2，3，4。。。递增，OK没有问题，有一个引用，就不会释放

## 再测试一下删除引用后的总数
```java
package wanghuida.test;

import java.util.WeakHashMap;
import java.util.Map;

import java.util.List;
import java.util.ArrayList;

public class Entry {

    public static void main(String[] args) {
        List<String[]> templist = new ArrayList<String[]>();  
        //设的多一点，可以让GC真实发挥
        for(int i=0; i < 1000; i++){
            String[] tempstr = new String[2];
            templist.add(tempstr);
        }
        
        Map<String[], String[]> map = new WeakHashMap<String[], String[]>();
        for(int i=0; i < 100; i++){
            map.put(templist.get(i), new String[2]);
            templist.set(i, null); //删除掉引用 
            System.gc();
            System.out.println(map.size());
        }

    }

}
```

输出0，1，0，1，1，1。。。保持下去，OK也没问题，删除引用后就会释放

## 再测试一下有引用但没有使用的情况
```java
package wanghuida.test;

import java.util.WeakHashMap;
import java.util.Map;

import java.util.List;
import java.util.ArrayList;

public class Entry {

    public static void main(String[] args) {
        List<String[]> templist = new ArrayList<String[]>();  
        //新增一个引用
        List<String[]> list = new ArrayList<String[]>();  
        //设的多一点，可以让GC真实发挥
        for(int i=0; i < 1000000; i++){
            String[] tempstr = new String[2];
            templist.add(tempstr);
            list.add(tempstr);
        }
        
        Map<String[], String[]> map = new WeakHashMap<String[], String[]>();
        for(int i=0; i < 100; i++){
            map.put(templist.get(i), new String[2]);
            templist.set(i, null); //删除掉引用 
            System.gc();
            System.out.println(map.size());
        }

    }

}
```
输出0，1，0，1，1，1。。。保持下去，OK也没问题，有引用但不使用也就会释放
## 最后对比一下有使用的情况
```java
package wanghuida.test;

import java.util.WeakHashMap;
import java.util.Map;

import java.util.List;
import java.util.ArrayList;

public class Entry {

    public static void main(String[] args) {
        List<String[]> templist = new ArrayList<String[]>();  
        //新增一个引用
        List<String[]> list = new ArrayList<String[]>();  
        //设的多一点，可以让GC真实发挥
        for(int i=0; i < 1000000; i++){
            String[] tempstr = new String[2];
            templist.add(tempstr);
            list.add(tempstr);
        }
        
        Map<String[], String[]> map = new WeakHashMap<String[], String[]>();
        for(int i=0; i < 100; i++){
            map.put(templist.get(i), new String[2]);
            templist.set(i, null); //删除掉引用 
            System.gc();
            System.out.println(map.size());
        }
        
        System.out.println(list.size());

    }

}
```
输出1，2，3，4。。。递增，OK了