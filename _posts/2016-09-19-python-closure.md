---
layout: post
title: Python - 闭包
categories: [Python]
description: Understanding Python's Closure.
keywords: Python, Closure
---

#### 闭包

高阶函数除了可以最为参数外，还可以作为函数的返回值。我们来看下一个求和函数的例子：

```python
def get_sum(*args):
    ax = 0
    for arg in args:
        ax = ax + arg
    return ax
```

如果需求变化，不需要立即返回求和结果，而是后续根据需要再计算，要怎么实现？这就需要不返回求和结果，而是返回函数了：

```python
def lazy_get_sum(*args):
    def get_sum():
        ax = 0
        for arg in args:
            ax = ax + arg
        return ax
    return get_sum
```

当我们调用 `lazy_get_sum` 时，返回的并不是求和结果，而是求和函数：

```bash
>>> f = lazy_get_sum(1,2,3)
>>> f
<function get_sum at 0x7f1c647abb90>
```

只有在调用 `f` 函数时，才真正计算求和：

```bash
>>> f()
6
>>>
```

在这个例子中，函数 'lazy_get_sum' 内部又定义了一个函数 `get_sum`，并且内部函数 `get_sum` 使用了外部函数 `lazy_get_sum` 的局部变量。当 `lazy_get_sum` 返回 `get_sum` 时，相关参数和局部变量都保持在返回的函数中，这种程序结果成为"闭包(Closure)"。其实"闭包"就是作为返回值返回的函数，并且在需要的时候再调用该函数，不必马上执行。

每次我们调用 `lazy_get_sum` 函数时，返回的都是一个新函数，即使传入的是相同的参数：

```bash
>>> f1 = lazy_get_sum(1,2,3)
>>> f2 = lazy_get_sum(1,2,3)
>>> f1 == f2
False
>>>
```

f1 和 f2 的调用结果互相不产生影响。

注意一点，返回的内部函数引用了外部函数的局部变量。所以，即使外部函数执行完毕返回了，其内部变量还会被内部函数所引用。但是，如果内部函数引用了外部函数的循环变量，可能会产生意外的结果。我们来看下这个例子：

```python
def count():
    rs = []
    for i in xrange(1, 5):
        def f():
            return i*i
        rs.append(f)
    return rs

f1, f2, f3, f4 = count()
```

上面例子中，每一次循环都创建一个新的函数。然后将返回的4个函数分别赋值给f1, f2, f3, f4。但是，这四个函数的执行结果却不是预想中的1，4，9，16，实际结果是：

```bash
>>> f1, f2, f3, f4 = count()
>>> f1()
16
>>> f2()
16
>>> f3()
16
>>> f4()
16
```

原因就是这四个函数不是返回时立即执行的，等到具体执行时，局部变量`i`的值经过循环已经变为4，所以4个函数的执行结果都是16。这个例子告诉我们，使用闭包，在返回函数中不要引用循环变量或者后续会发生变化的变量。

如果一定要引用循环变量，我们该如何实现呢：

```python
def count():
    rs = []
    for i in xrange(1, 5):
        def f(j):
            def g():
                return j*j
            return g
        rs.append(f(i))
    return rs

f1, f2, f3, f4 = count()
```

```bash
>>> f1()
1
>>> f2()
4
>>> f3()
9
>>> f4()
16
```

以上是关于"闭包"的基础概念介绍，和基础使用举例。关于"闭包"还有其他内容，留待以后补充。

#### 补充
