---
layout: post
title: Python 匿名函数
categories: [Python]
description: lambda in Python
keywords: Python, lambda, 匿名函数
---

Python 使用关键字 lambda 创建匿名函数，所谓匿名，意即不再使用 def 语句这样标准的形式定义一个函数。这种语句的目的是由于性能的原因，在调用时绕过函数的栈分配。其语法是：

```python
lambda [arg1[, arg2, ... argN]]: expression
```

以 map() 为例，直接将匿名函数作为参数传递：

```python
map(lambda x: x*x, [1,2,3,4,5])
[1,4,9,16,25]
```

lambda x: x*x 实际上相当于：

```python
def func(x):
    return x*x
```

```python
func = lambda x:x*x
func
<function <lambda> at 0x20a38c0>
func(5)
25
```

可以将匿名函数作为返回值：

```python
def func(x):
    return lambda x:x*x

func(5)
<function <lambda> at 0x20a38c0>
```
