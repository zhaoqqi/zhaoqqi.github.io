---
layout: post
title: Python的 *args 和 **kwargs
categories: [Python]
description: Python *args and **kwargs
keywords: Python, args, kwargs
---

##### 分析 *args

当你不确定函数在使用时会有多少个参数传入时，可以使用[*args]定义函数的参数，args类型为 tuple。

```python
def print_args(*args):
    print type(args)
    print args
    for count, key in enumerate(args):
        print '{0}. {1}'.format(count, key)

>>> print_args('apple', 'banana', 'others')
<type 'tuple'>
('apple', 'banana', 'others')
0. apple
1. banana
2. others
```

##### 分析 **kwargs

与[*args]相似，[**kwargs]在你不知道传入参数数量的时候使用，kwargs 类型为 dict。
额外功能是，还可以指定参数的名称。

```python
def print_kwargs(**kwargs):
    print type(kwargs)
    print kwargs
    for key, value in kwargs.items():
        print '{0} = {1}'.format(key, value)

>>> print_kwargs(apple='fruit', cabbage='vegetable')
<type 'dict'>
{'cabbage': 'vegetable', 'apple': 'fruit'}
cabbage = vegetable
apple = fruit
```
