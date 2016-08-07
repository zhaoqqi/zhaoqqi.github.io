---
layout: post
title: Python generators and yield
categories: [Python]
description: Python generator and yield
keywords: Python, generator, yield
---

  Python generators 可以简单而有效的实现迭代功能。Generators 看起来有点像是 functions。
在语法上，他们之间还是存在差异的。Functions 使用 return 语句结束处理，generators使用
yield语句代表每次迭代的返回值，每个 generators 可以使用一个或者多个 yield 语句返回。

  Python generators 一个重要特性是，在两次调用之间，generator function 的局部变量和下次调用
开始执行的起点会保存下来。普通 function 每次调用都从起始位置开始执行，局部变量重新生成并且
被赋值。Generator functions 被调用时，会从上次执行到的 yield 语句之后的位置开始执行，并且内部
的局部变量也会被保存。

  Python generators 迭代完所有成员后，会 raise StopIteration 退出迭代。
Generators 不能使用 reset 方法重新设置迭代的起点，只能重新生成 generator，从头开始迭代。

先来个简单的 generators 的例子：

```python
def fc_generator():
    yield('Arsenal FC')
    yield('Livpool FC')
    yield('Man City FC')
    yield('SwanSea City FC')
```

```sh
# 输出结果：
>>> gen = fc_generator()
>>> gen.next()
'Arsenal FC'
>>> gen.next()
'Livpool FC'
>>> gen.next()
'Man City FC'
>>> gen.next()
'SwanSea City FC'
>>> gen.next()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
>>>
```

提到 generators，不得体免俗的提到斐波那契数列的例子 : )

```python
def fibonacci(n):
    """ fibonacci numbers generator """
    a, b, counter = 0, 1, 0
    while True:
        if counter < n:
            yield b
            a, b = b, a+b
            counter += 1
        if counter >= n:
            break

if __name__ == '__main__':
    f = fibonacci(5)
    for num in f:
        print num
```

Generators 也可以使用类实现，该类要实现iterator的功能，需要实现自己的 \_\_iter\_\_() 和 next() 方法：

```python
class Fib(object):
    def __init__(self, max):
        self.max = max
        self.a, self.b, self.counter = 0, 1, 0

    def __iter__(self):
        return self

    def next(self):
        if self.counter < self.max:
            ret = self.b
            self.a, self.b = self.b, self.a+self.b
            self.counter +=1
            return ret
        raise StopIteration

for num in Fib(5):
    print num
```

对比 generator function，用类实现生成器迭代功能要繁琐不少。
