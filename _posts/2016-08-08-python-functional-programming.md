---
layout: post
title: 函数式编程
categories: [Python]
description: Functional programming in Python
keywords: Python, functional programming, 函数式编程
---

最近看了几篇关于 Python 函数式编程的 Blog，犹如一捧甘甜的清泉入口，甘冽清爽，怡人肺腑。
有点喜欢上函数式编程的这种清爽简洁。尤其是陈皓的酷壳中的函数式编程这篇博文，本文引用很多该文的内容，特此说明。

那么什么是函数式编程呢？先罗列一下函数编程的几条基本原则：

1. 函数在处理过程中不使用局部变量，不修改或者保存外部变量的值或状态。
2. 函数的处理过程和返回结果具有确定性，也就是说在输入一定的情况下，输出也是确定的。
3. 函数可以作为另一个函数的参数，也可以赋值给变量，作为普通变量使用。
4. 函数式编程注重的是描述问题而不是具体如何实现。

##### 函数式与非函数式的简单对比

还是从简单的例子入手，对比一下函数式与非函数式最直观的差别：

```python
# 非函数式
step = 5
def decrease():
    step -= 1

# 函数式
step = 5
def decrease():
    return step-1
```

函数式不依赖于外部数据，不修改外部数据的状态，只是经过计算返回一个新的值。

##### 函数当作变量使用

```python
def multiplier(x):
    def multiplicand(y):
        return x*y
    return multiplicand

mul5 = multiplier(5)
mul6 = multiplier(6)

>>> print mul5(6)
30
>>> print mul6(5)
30
```

##### Python 提供的函数式编程的便利工具

在函数式编程中，我们不应该用普通的循环迭代的方式，而应该使用更高效易用的方式。
Python 为我们提供了典型的函数式编程的工具函数，以及 lambda 表达式。

###### map/reduce

map()函数接受两个参数，一个是函数，一个是iterable的对象。map将传入的函数依次作用到序列的每个元素，
并将结果作为新的Iterator返回。

```python
clubs = ['Arsenal', 'Swansea', 'Liverpool']
clubs_n = map(len, clubs)
print clubs_n
```

使用自定义函数

```python
def upper(item):
    return item.upper()

clubs = ['Arsenal', 'Swansea', 'Liverpool']
club_n = map(upper, clubs)
print clubs_n
```

使用lambda

```python
def upper(item):
    return item.upper()

clubs = ['Arsenal', 'Swansea', 'Liverpool']
club_n = map(lambda x: upper(x), clubs)

>>> print club_n
['ARSENAL', 'SWANSEA', 'LIVERPOOL']
```

###### reduce

reduce()把一个函数作用在一个序列[x1, x2, x3...]上，这个函数必须接收两个参数，reduce把结果继续和序列的下一个元素做累积计算，其效果就是：

```python
>>> reduce(f, [x1, x2, x3,x4]) = f(f(f(x1, x2), x3), x4)
```

求一个序列和的例子：

```python
def add(x, y):
    return x + y

>>> sum = reduce(add, [1,1,2,3,5,8,13,21,34])
>>> print sum
88
```

使用lambda

```python
def add(x, y):
    return x + y

>>> sum = reduce(lambda x,y: add(x,y), [1,1,2,3,5,8,13,21,34])
>>> print sum
88
```

###### filter

和map()类似，filter()也接收一个函数和一个序列。和map()不同的时，filter()把传入的函数依次作用于每个元素，然后根据返回值是True还是False决定保留还是丢弃该元素。

```python
def even(x):
    if x%2 == 0:
        return True

>>> even_list = filter(even, [1,2,3,4,5,6,7,8])
>>> print even_list
[2, 4, 6, 8]
```

使用lambda

```python
def even(x):
    if x%2 == 0:
        return True

>>> even_list = filter(lambda x: even(x), [1,2,3,4,5,6,7,8])
>>> print even_list
[2, 4, 6, 8]
```

###### sorted

sorted()用来对一个Iterator的序列进行排序。通常规定，对于两个元素x和y，如果认为x < y，则返回-1，如果认为x == y，则返回0，如果认为x > y，则返回1，这样，排序算法就不用关心具体的比较过程，而是根据比较结果直接排序。

```python
>>> sorted([23,5,16,9,5])
[5, 5, 9, 16, 23]
```

此外，sorted()是高阶函数，接受一个比较函数实现自定义的排序。比如，想倒序排列上一个实例的list：

```python
def inverted_order(x, y):
    if x > y:
        return -1
    if x < y:
        return 1
    if x == y:
        return 0

>>> sorted([23,5,16,9,5], inverted_order)
[23, 16, 9, 5, 5]
```

使用lambda

```python
# On Python 3.x:  a = sorted(a, key=lambda x: x.modified, reverse=True)
# On Python 2.x:  sorted(iterable, cmp=None, key=None, reverse=False)
>>> sorted([23,5,16,9,5], cmp=inverted_order, key=None, reverse=True)
[5, 5, 9, 16, 23]
```

除了 map/reduce, filter, sorted外，Python 还有 find, all, any 等高阶辅助函数。

##### Pipeline

以下内容（原封不动）来自：http://coolshell.cn/articles/10822.html

pipeline 管道借鉴于Unix Shell的管道操作——把若干个命令串起来，前面命令的输出成为后面命令的输入，如此完成一个流式计算。（注：管道绝对是一个伟大的发明，他的设哲学就是KISS – 让每个功能就做一件事，并把这件事做到极致，软件或程序的拼装会变得更为简单和直观。这个设计理念影响非常深远，包括今天的Web Service，云计算，以及大数据的流式计算等等）

比如，我们如下的shell命令：

```sh
ps auwwx | awk '{print $2}' | sort -n | xargs echo
```

也可以类似下面这个样子：

```python
pids = for_each(result, [ps_auwwx, awk_p2, sort_n, xargs_echo])
```

好了，让我们来看看函数式编程的Pipeline怎么玩？
我们先来看一个如下的程序，这个程序的process()有三个步骤：
1. 找出偶数。
2. 乘以3
3. 转成字符串返回

```python
def process(num):
    # filter out non-evens
    if num % 2 != 0:
        return
    num = num * 3
    num = 'The Number: %s' % num
    return num
 
nums = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
 
for num in nums:
    print process(num)
 
# 输出：
# None
# The Number: 6
# None
# The Number: 12
# None
# The Number: 18
# None
# The Number: 24
# None
# The Number: 30
```

我们可以看到，输出的并不够完美，另外，代码阅读上如果没有注释，你也会比较晕。下面，我们来看看函数式的pipeline（第一种方式）应该怎么写？

```python
def even_filter(nums):
    for num in nums:
        if num % 2 == 0:
            yield num
def multiply_by_three(nums):
    for num in nums:
        yield num * 3
def convert_to_string(nums):
    for num in nums:
        yield 'The Number: %s' % num
 
nums = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
pipeline = convert_to_string(multiply_by_three(even_filter(nums)))
for num in pipeline:
    print num
# 输出：
# The Number: 6
# The Number: 12
# The Number: 18
# The Number: 24
# The Number: 30
```

我们动用了Python的关键字 yield，这个关键字主要是返回一个Generator，yield 是一个类似 return 的关键字，只是这个函数返回的是个Generator-生成器。所谓生成器的意思是，yield返回的是一个可迭代的对象，并没有真正的执行函数。也就是说，只有其返回的迭代对象被真正迭代时，yield函数才会正真的运行，运行到yield语句时就会停住，然后等下一次的迭代。（这个是个比较诡异的关键字）这就是lazy evluation。

好了，根据前面的原则——“使用Map & Reduce，不要使用循环”，那我们用比较纯朴的Map & Reduce吧。

```python
def even_filter(nums):
    return filter(lambda x: x%2==0, nums)
 
def multiply_by_three(nums):
    return map(lambda x: x*3, nums)
 
def convert_to_string(nums):
    return map(lambda x: 'The Number: %s' % x,  nums)
 
nums = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
pipeline = convert_to_string(
               multiply_by_three(
                   even_filter(nums)
               )
            )
for num in pipeline:
    print num
```

但是他们的代码需要嵌套使用函数，这个有点不爽，如果我们能像下面这个样子就好了（第二种方式）。

```python
pipeline_func(nums, [even_filter,
                     multiply_by_three,
                     convert_to_string])
```

那么，pipeline_func 实现如下：

```python
def pipeline_func(data, fns):
    return reduce(lambda a, x: x(a),
                  fns,
                  data)
```
