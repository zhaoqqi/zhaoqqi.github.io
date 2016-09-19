---
layout: post
title: Python 装饰器
categories: [Python]
description: Python decorator
keywords: Python, decorator
---

##### 先看个例子

```python
def hello(func):
    print 'hello func() ...'
    func()
    print 'goodbye func() ...'

@hello
def func():
    print 'I am func() .'

# output:
hello func() ...
I am func() .
goodbye func() ...
```

从这个例子看来，真正执行的是hello函数，func函数作为参数传给了hello函数。
那么直接执行hello(func)会怎么样呢？

```python
>>> hello(func)
hello func() ...
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<stdin>", line 3, in hello
TypeError: 'NoneType' object is not callable
```

看来程序将hello(func)的返回值当作一个函数来执行了，然而我们的hello函数并没有设置返回值……
我们来改进一下：

```python
def hello(func):
    def wrapper():
        print 'hello func() ...'
        func()
        print 'goodbye func() ...'
    return wrapper

def func():
    print 'I am func.'

>>> hello(func)
<function wrapper at 0x0000000002255C88>

>>> hello(func)()
hello func() ...
I am func.
goodbye func() ...
```

这时，hello(func)的返回值就成为wrapper()函数，这时再执行hello(func)()就相当于执行wrapper()函数。
装饰器本质上就是被装饰的函数当作一个参数，被传入装饰器函数执行。在执行函数功能之前或之后，
可以利用装饰器，执行函数功能之外的任务。比如，资源状态的校验，残留资源的清理等。

##### 带参数的装饰器

```python
def sayHelloTo(name):
    def real_proc_wrapper(func):
        def wrapper():
            return '%s %s, nice to meet you.' % (func(), name)
        return wrapper
    return real_proc_wrapper

@sayHelloTo(name='Tom')
def hello():
    return 'Hello'

# output
print hello()
Hello Tom, nice to meet you.
```

最外层的sayHelloTo()函数用来处理装饰器的参数传入。
real_proc_wrapper()才是真正的装饰器函数，传入被装饰的函数func()作为参数。


##### 使用类装饰器
先上个简单例子，看清楚在装饰过程中的代码执行顺序

```python
class Hello(object):
    
    def __init__(self, func):
        print 'in Hello __init__().'
        self.func = func

    def __call__(self):
        print 'in Hello __call__().'
        self.func()

@Hello
def func():
    print 'in func() ...'
    print 'hello'

# output
in Hello __init__().
>>> print 'finished decorating func() ...'
finished decorating func() ...
>>> func()
in Hello __call__().
in func() ...
hello
```

##### 带参数的类装饰器

```python
class Hello(object):
    
    def __init__(self, name):
        self.name = name

    def __call__(self, func):
        def wrapper(name):
            return '%s %s, nice to meet you.' % (func(), name)
        return wrapper

@Hello(name='Tom')
def func():
    return 'Hello'

>>> print hello()
Hello Tom, nice to meet you.
```

有一点要注意：如果类装饰器有参数传入，func就不能从\_\_init\_\_()函数传入，而要从\_\_call\_\_()函数传入。


##### 装饰器的副作用
我们可以看到，在本文第二个示例中，func()函数被装饰过以后返回的函数是wrapper()，而不是我们期望的func()：

```python
>>> hello(func)
<function wrapper at 0x0000000002255C88>
```

为了消除这个隐患可能带来的副作用，Python提供了functools包中叫wraps的decorator。
下面是改进后的示例：

```python
from functools import wraps

def hello(func):
    @wraps(func)
    def wrapper():
        print 'hello func() ...'
        func()
        print 'goodbye func() ...'
    return wrapper

def func():
    print 'I am func.'

>>> hello(func)
<function func at 0x0000000002255EB8>

>>> hello(func)()
hello func() ...
I am func.
goodbye func() ...
```
