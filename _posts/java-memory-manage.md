---
title: JVM内存管理-垃圾回收与内存分配
link_title: java-memory-manage
date: 2017-08-14 14:22:40
tags: [Java, JVM]
categories: Java
thumbnailImage: http://onxkn9cbz.bkt.clouddn.com/34.jpg	
thumbnailImagePosition: left
---
<span/>
<!-- more -->
![](http://onxkn9cbz.bkt.clouddn.com/34.jpg)
<!-- toc -->
> Java技术体系中所提倡的自动内存管理最终可以归结为自动化地解决了两个问题：给对象分配内存以及回收分配给对象的内存。

# Java垃圾收集

## 哪些内存需要回收？
线程私有区的程序计数器、虚拟机栈和本地方法栈不需要，重点是共享数据区的堆和方法区部分的内存

## 什么时候回收？
### 判断对象是否存活的算法？
**引用计数法**

逻辑：给对象添加一个引用计数器，每当有一个地方引用它时，计数器值就加1，当引用失效时，计数器值就减1，任何时刻计数器为0的对象就是不可能再被使用的。

优点：实现简单，效率高

缺点：没有解决互相循环引用问题

Java虚拟机并没有选择这种算法来进行垃圾回收

**可达性分析算法**

逻辑：这种算法的基本思路是通过一系列名为“GC Roots”的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链，当一个对象到GC Roots没有任何引用链相连时，就证明此对象是不可用的。

Java语言是通过可达性分析算法来判断对象是否存活的。

在Java语言里，可作为GC Roots的对象包括下面几种：

虚拟机栈（栈帧中的本地变量表）中引用的对象。
方法区中的类静态属性引用的对象。
方法区中的常量引用的对象。
本地方法栈中JNI（Native方法）的引用对象。

### 正确理解引用，java对象有哪几种引用类型？
强引用，软引用，弱引用，虚引用

### 对死亡的标记过程
即使在可达性分析算法中不可达的对象，也并非是“非死不可”的，这时候它们暂时处于“缓刑”阶段，要真正宣告一个对象死亡，至少要经历两次标记过程。

### 回收方法区
很多人认为方法区（或者HotSpot虚拟机中的永久代）是没有垃圾收集的，Java虚拟机规范中确实说过可以不要求虚拟机在方法区实现垃圾收集，而且在方法区中进行垃圾收集的“性价比”一般比较低：在堆中，尤其是在新生代中，常规应用进行一次垃圾收集一般可以回收70%～95%的空间，而永久代的垃圾收集效率远低于此

永久代的垃圾收集主要回收两部分内容：废弃常量和无用的类。

## 如何回收？
### 有哪些回收算法？
- 标记-清除算法

算法分为“标记”和“清除”两个阶段：首先标记出所有需要回收的对象，在标记完成后统一回收所有被标记的对象。
缺点：效率低,标记清除后会产生大量不连续的内存，可能会导致以后程序程序运行过程中需要分配较大对象时，无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作。

- 复制算法

它将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。
优点：简单高效
缺点：代价是将内存缩小为原来的一半，代价高

现在的商业虚拟机都采用这种收集算法来回收新生代，研究表明，新生代中的对象98%是“朝生夕死”的，所以并不需要按照1∶1的比例来划分内存空间，而是将内存分为一块较大的Eden空间和两块较小的Survivor空间，每次使用Eden和其中一块Survivor。
当回收时，将Eden和Survivor中还存活着的对象一次性地复制到另外一块Survivor空间上，最后清理掉Eden和刚才用过的Survivor空间。HotSpot虚拟机默认Eden和Survivor的大小比例是8∶1，也就是每次新生代中可用内存空间为整个新生代容量的90%，只有10%的内存会被“浪费”。
当然，90%的对象可回收只是一般场景下的数据，我们没有办法保证每次回收都只有不多于10%的对象存活，当Survivor空间不够用时，需要依赖其他内存（这里指老年代）进行分配担保（Handle Promotion）。


- 标记-整理算法

标记过程仍然与“标记-清除”算法一样，但后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存。

- 分代收集算法

当前商业虚拟机的垃圾收集都采用“分代收集”（Generational Collection）算法，这种算法并没有什么新的思想，只是根据对象存活周期的不同将内存划分为几块。一般是把Java堆分为新生代和老年代，这样就可以根据各个年代的特点采用最适当的收集算法。

在新生代中，每次垃圾收集时都发现有大批对象死去，只有少量存活，那就选用复制算法，只需要付出少量存活对象的复制成本就可以完成收集。

在老年代中因为对象存活率高、没有额外空间对它进行分配担保，就必须使用"标记—清理"或者"标记—整理"算法来进行回收。


# Java垃圾收集器
## 概念理解
- 并发和并行

并行（Parallel）：指多条垃圾收集线程并行工作，但此时用户线程仍然处于等待状态。

并发（Concurrent）：指用户线程与垃圾收集线程同时执行（但不一定是并行的，可能会交替执行），用户程序在继续运行，而垃圾收集程序运行于另一个CPU上。

- Minor GC 和 Full GC？

新生代GC（Minor GC）：指发生在新生代的垃圾收集动作，因为Java对象大多都具备朝生夕灭的特性，所以Minor GC非常频繁，一般回收速度也比较快。

老年代GC（Major GC / Full GC）：指发生在老年代的GC，出现了Major GC，经常会伴随至少一次的Minor GC（但非绝对的，在Parallel Scavenge收集器的收集策略里就有直接进行Major GC的策略选择过程）。Major GC的速度一般会比Minor GC慢10倍以上。

## 收集器
### Serial收集器
- 最基本、发展历史最悠久的收集器
- 单线程，必须暂停其他所有的工作线程，直到它收集结束
- Serial收集器是虚拟机运行在Client模式下的默认新生代收集器。
- 简单高效

### ParNew收集器
- Serial收集器的多线程版本，除了使用多条线程进行垃圾收集之外其他和Serial收集器完全一样

### Parallel Scavenge收集器
- 是一个新生代收集器，它也是使用复制算法的收集器，又是并行的多线程收集器。

### Serial Old收集器
- Serial Old是Serial收集器的老年代版本，它同样是一个单线程收集器，使用标记－整理算法。

### Parallel Old收集器
Parallel Old是Parallel Scavenge收集器的老年代版本，使用多线程和“标记－整理”算法。

### CMS收集器
CMS收集器是基于“标记—清除”算法实现的，从总体上来说，CMS收集器的内存回收过程是与用户线程一起并发执行的。

### G1收集器
并行与并发，分代收集，	空间整合，可预测的停顿

# Java对象内存分配策略
本文中的内存分配策略指的是Serial / Serial Old收集器下（ParNew / Serial Old收集器组合的规则也基本一致）的内存分配和回收的策略

## 对象优先在Eden分配
大多数情况下，对象在新生代Eden区中分配。当Eden区没有足够空间进行分配时，虚拟机将发起一次Minor GC。

## 大对象直接进入老年代
### -XX:PretenureSizeThreshold参数
虚拟机提供了一个-XX:PretenureSizeThreshold参数，令大于这个设置值的对象直接在老年代分配。这样做的目的是避免在Eden区及两个Survivor区之间发生大量的内存复制（复习一下：新生代采用复制算法收集内存）。

### 长期存活的对象将进入老年代

对象年龄的判定:
如果对象在Eden出生并经过第一次Minor GC后仍然存活，并且能被Survivor容纳的话，将被移动到Survivor空间中，并且对象年龄设为1。
对象在Survivor区中每“熬过”一次Minor GC，年龄就增加1岁，当它的年龄增加到一定程度（默认为15岁），就将会被晋升到老年代中。
对象晋升老年代的年龄阈值，可以通过参数-XX:MaxTenuringThreshold设置。

## 空间分配担保
在发生Minor GC之前，虚拟机会先检查老年代最大可用的连续空间是否大于新生代所有对象总空间，如果这个条件成立，那么Minor GC可以确保是安全的。如果不成立，则虚拟机会查看HandlePromotionFailure设置值是否允许担保失败。如果允许，那么会继续检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小，如果大于，将尝试着进行一次Minor GC，尽管这次Minor GC是有风险的；如果小于，或者HandlePromotionFailure设置不允许冒险，那这时也要改为进行一次Full GC。

（本文完）

整理自：
- http://www.jianshu.com/p/424e12b3a08f
- http://www.jianshu.com/p/50d5c88b272d

