---
layout: post
title: Python - 对象的内存使用
categories: [Python]
description: Python memery object.
keywords: Python
---

#### 动态类型

所谓动态、静态指的是变量的绑定方式。静态语言在编译时绑定，动态语言可以在运行时随意绑定。Python 属于动态类型语言。

#### 强类型与若类型

强类型、若类型指的是变量的类型在运算的上下文中是否可以自动转换。强类型可以，Python 属于强类型。

#### 赋值语句

我们从一个简单的赋值语句开始观察 Python 的对象在内存中的使用情况：

```python
a = 1
```

整数 1 为一个对象，而 a 是一个对象 a 的引用。Python 是动态类型的编程语言，对象与引用是分离的。下面我们使用 Python 内置函数 `id()` 作为辅助，来查看对象的内存地址：

```python
>>> a = 1
>>> print(id(a))
37987128
>>> print(hex(id(a)))
'0x243a338'
```

`id(a)` 返回10进制内存地址，`hex(id(a))` 返回16进制内存地址。

#### 内存对象和引用

在 Python 中，整数和短小的字符，Python 都会缓存这些对象，以便重复使用。当我们赋值(创建)多个等于1的对象的引用时，实际上这些引用都指向同一个对象。

```python
>>> a = 1
>>> b = 1
>>> print(hex(id(a)))
0x243a338
>>> print(hex(id(b)))
0x243a338
>>> print(hex(id(1)))
0x243a338
```

可见，a 和 b 均指向同一个对象 1 的内存地址。

下面我们再看看，其他 Python 类型的对象：

```python
>>> a = 1
>>> b = 1
>>> print(a is b)
True

>>> a = 'hello'
>>> b = 'hello'
>>> print(a is b)
True

>>> a = 'hello world'
>>> b = 'hello world'
>>> print(a is b)
False

>>> a = []
>>> b = []
>>> print(a is b)
False

>>> a = 100.002
>>> b = 100.002
>>> print(a is b)
False
```

可以看到，Python 只缓存了整型和短字符串，因此每个对象在内存中只存一份。比如，所有整数1的引用都指向同一个内存对象。即使使用赋值语句，也只是创建了对象新的引用，而不是创建新的内存对象。长字符串和其他类型可以有多个相同的对象，可以使用赋值语句创建新的内存对象。
