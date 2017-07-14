---
title: 《算法图解》书摘-递归/快速排序
link_title: algorithm-illustration-02
date: 2017-05-08 14:41:46
tags: [算法图解]
categories: 算法
thumbnailImage: http://onxkn9cbz.bkt.clouddn.com/15.jpg	
thumbnailImagePosition: left
---
<!-- toc -->
<!-- more -->
![](http://onxkn9cbz.bkt.clouddn.com/15.jpg)
# 第三章 递归
- 递归只是让解决方案更清晰，并没有性能上的优势。实际上，在有些情况下，使用循环的性能更好。

- “如果使用循环，程序的性能可能更高；如果使用递归，程序可能更容易理解。如何选择要看什么对你来说更重要。”

- 编写递归函数时，必须告诉它何时停止递归。正因为如此，每个递归函数都有两部分：基线条件（base case）和递归条件（recursivecase）。递归条件指的是函数调用自己，而基线条件则指的是函数不再调用自己，从而避免形成无限循环。

## 小结
- 递归指的是调用自己的函数。
- 每个递归函数都有两个条件：基线条件和递归条件。
- 栈有两种操作：压入和弹出。
- 所有函数调用都进入调用栈。
- 调用栈可能很长，这将占用大量的内存。

# 第四章 快速排序
- 我们将探索分而治之
（divide and conquer，D&C）——一种著名的递归式问题解决方法。你将学习第一个重要的D&C算法——快速排序

- 快速排序代码
```python
def quicksort(array)：
  if len(array) < 2:
    return array
  else:
    pivot = array[0]
    less = [i for i array[1:] if i <= pivot]
    greater = [i for i in array[i:] if i > pivot]
    return quicksort(less) + [pivot] + quicksort(greater)
  print quicksort([10,5,2,3])
```


- 整个算法需要的时间为O(n) * O(log n) = O(n log n)。这就是最佳情况。
在最糟情况下，有O(n)层，因此该算法的运行时间为O(n) * O(n) = O(n2)。
知道吗？这里要告诉你的是，最佳情况也是平均情况。只要你每次都随机地选择一个数组元
素作为基准值，快速排序的平均运行时间就将为O(n logn)。快速排序是最快的排序算法之一

## 小结
- D&C将问题逐步分解。使用D&C处理列表时，基线条件很可能是空数组或只包含一个元
素的数组。
- 实现快速排序时，请随机地选择用作基准值的元素。快速排序的平均运行时间为O(n log n)。
- 大O表示法中的常量有时候事关重大，这就是快速排序比合并排序快的原因所在。
- 比较简单查找和二分查找时，常量几乎无关紧要，因为列表很长时，O(log n)的速度比O(n)快得多。

