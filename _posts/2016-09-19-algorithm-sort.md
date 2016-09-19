---
layout: post
title: 经典排序算法的 Python 实现
categories: [algorithm]
description: Eight classic sort algorithms with Python.
keywords: Python, algorithm
---

#### 插入排序

##### 描述
插入排序是通过内外双重循环实现。外部循环按照序列顺序从头到尾依次循环取每个值key，内部循环倒序取值序列中key之前的每个元素。比如，x 为 key 之前的一个元素，如果 x < key ，则交换两者的位置，直到对比交换到第0个元素。插入排序使用双重循环，所以时间复杂度为O(n^2)。是稳定的排序算法。

##### 代码实现

```python
def insert_sort(lists):
    count = len(lists)
    for i in range(1, count):
        key = lists[i]
        j = i - 1
        while j >= 0:
            if lists[j] > key:
                lists[j+1] = list[j]
                lists[j] = key
            j -= 1
    return lists
```
