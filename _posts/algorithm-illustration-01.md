---
title: 《算法图解》书摘-算法介绍/选择排序
link_title: algorithm-illustration-01
date: 2017-05-03 14:11:53
tags: [算法]
categories: 书摘
thumbnailImage: https://upload-images.jianshu.io/upload_images/79431-5f282848266cf082.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/509/format/webp	
thumbnailImagePosition: left
---
<!-- toc -->
<!-- more -->
![](https://upload-images.jianshu.io/upload_images/79431-5f282848266cf082.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/509/format/webp)

# 第一章 算法介绍

## 二分查找Python代码
```python
def binary_search(list,item):
    low = 0
    high = len(list-1)
    
    while low <= high:
        mid =(low+high)
        guess = list[mid]
        if guess == item:
            return mid
        if guess > item：
            high = mid -1
        else:
            low = mid + 1
        return None    
```

## 小结
- 二分查找的速度比简单查找快得多
- O(log n)比O(n)快，需要搜索的元素越多，前者比后者就快得越多
- 算法运行时间并不是以秒为单位
- 算法运行时间是从其增速的角度度量的
- 算法运行时间用大O表示法表示

# 第二章 选择排序
- 很多算法仅在数据经过排序后才管用.选择排序是下一章介绍的快速排序的基石
- 数组和链表操作运行时间

#|数组 |链表
---|---|---
读取 | o(1)|o(n)
输入 | o(n)|o(1)
删除|o(n)|o(1)

- 需要随机读取元素时，数组效率高，当需要中间插入元素时，链表是更好的选择，链表只能随机访问

- 选择排序，需要的总时间是O(nxn)

- 示例代码
```python
def findSmallest(arr):
    smallest = arr[0]
    smallest_index = 0
    for i in range(1,len(arr)):
        if arr[i] < smallest;
            smallest = arr[i]
            smallest_index = i
    return smallest_index
    
def selectionSort(arr):
 newArr = []
 for i in range(len(arr)):
    smallest = findSmallest(arr)
    newArr.append(arr.pop(smallest))
return newArr    
```
## 小结
- 计算机内存犹如一大堆抽屉。
- 需要存储多个元素时，可使用数组或链表。
- 数组的元素都在一起。
- 链表的元素是分开的，其中每个元素都存储了下一个元素的地址。
- 数组的读取速度很快。
- 链表的插入和删除速度很快。
- 在同一个数组中，所有元素的类型都必须相同（都为int、double等）。


