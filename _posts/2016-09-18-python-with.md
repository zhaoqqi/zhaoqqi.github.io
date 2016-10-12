---
layout: post
title: Python - with
categories: [Python]
description: Understanding Python's "with" statement.
keywords: Python, with
---

就像 Python 的其他语法一样，一旦你理解了要解决的问题，`with` 语句是非常简单的。看下面这个代码片段：

```python
    set things up
    try:
        do something
    finally:
        tear things down
```

如果 "set things up" 是打开一个文件，或者是获得一个外部资源。那么 "tear things down" 就是关闭文件，或者释放/删除资源。`try-finally` 结构确保 "tear things down" 总会被执行，甚至 "do something" 中途报错而没有执行完毕。

方便起见，你可以考虑将 "set things up" 和 "tear things down" 放到一个库函数中，从而可以重用。像这样：

```python
    def controlled_execution(callback):
        set things up
        try:
            callback(thing)
        finally:
            tear things down

    def my_function(thing):
        do something

    controlled_execution(my_function)
```

这有点罗嗦，尤其是在需要修改本地变量时。另一种实现方法是使用一次性生成器，使用 `for-in` 语句循环调用：

```python
    def controlled_execution():
        set things up
        try:
            yield thing
        finally:
            tear things down

    for thing in controlled_execution():
        do something with thing
```

但是在 Python2.4或更早的版本中，`yield` 语句是不允许写在 `try-finally` 结构中的。就算再2.5版本以后修复了该问题，在明知道只执行一次的情况下还使用循环结构还是很奇怪的。

所以，再考察了多种实现方式后，GvR 和 Python 开发小组实现了下面这种方式：

```python
    class controlled_execution:
        def __enter__(self):
            set things up
            return thing
        def __exit__(self, type, value, traceback):
            tear things down

    with controlled_execution() as thing:
         some code
```

现在，当 `with` 语句开始执行时，Python计算表达式的值，调用 `__enter__` 方法返回表达式的值(被称为"context guard")，使用 `as` 将 `__enter__` 方法的返回值赋值给一个变量。Python 接下来执行处理代码，无论何种情况发生，都会在最后调用 '__exit__'方法。

作为额外奖励，'__exit__' 方法可以查看抛出的异常，可以忽略或者选择处理异常。可以通过返回一个布尔值来忽略一个异常，就像这样：

```python
    def __exit__(self, type, value, traceback):
        return isinstance(value, TypeError)
```

在 Python2.7 中，文件对象就拥有 `__enter__` 和 `__exit__` 方法。前者返回文件对象本身，后者关闭文件对象：

```bash
    >>> f = open("/home/zhaoqi1/create_vpc_vm.sh")
    >>> f
    <open file '/home/zhaoqi1/create_vpc_vm.sh', mode 'r' at 0x7f1c64861d20>
    >>> f.__enter__()
    <open file '/home/zhaoqi1/create_vpc_vm.sh', mode 'r' at 0x7f1c64861d20>
    >>> f.read(1)
    '#'
    >>> f.__exit__(None,None,None)
    >>> f.read(1)
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    ValueError: I/O operation on closed file
```

所以，对于要打开一个文件，处理文件内容，然后确认关闭文件的操作来说，可以简单写成这样：

```python
    with open("/home/zhaoqi1/create_vpc_vm.sh") as f:
        data = f.read()
        print data
```
