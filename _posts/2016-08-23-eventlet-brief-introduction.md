---
layout: post
title: Eventlet(1) - 简介
categories: [OpenStack]
description: 先认识一下Eventlet.
keywords: OpenStack, Python, coroutines
---

##### Eventlet是什么

简单来说，Eventlet 是使用协程-coroutines(绿色线程-Green thread), 处理网络IO的非阻塞式的框架。

##### 重要概念

1. 协程-coroutines
    * 一个线程可以启动多个协程，多到成千上万。
    * 同一个时间点，只能有一个协程是执行状态，其他协程等待。
    * 同一个线程范围内的协程可以切换执行。
    * 它们共享线程的所有资源，因为互相之间没有冲突，而不用加锁。
2. 非阻塞式IO
    * 简单来说就是, 在处理多个IO请求的过程中，不会因为某个IO请求的读写没有就绪而一直处于等待状态，使其他IO请求无法处理。
    * 更详细的内容会参考APUE单开一篇文章进行描述。(TBD)

##### 绿化补丁

所谓绿化，即令系统网络模块中指定的模块协程友好(greenthread-friendly)。Python原生标准库不支持eventlet的协程，为此eventlet提供了patch。如果想在自己的应用程序中使用eventlet，就必须显示的绿化应用中要引入的模块。
同时设置socket为非阻塞式的。

使用Eventlet库的应用程序，必须使用以下两种方式之一，显示的执行绿化操作：

1. 通过从 `eventlet.green` 引入网络处理相关的库绿化应用程序
```python
from eventlet.green import socket
from eventlet.green import threading
from eventlet.green import asyncore
```

2. 使用 `monkeypatch` 在全局中绿化指定的系统模块
```python
'''该函数使用绿化后的对应函数，替换系统模块中对应的网络处理模块。
'''

eventlet.patcher.monkey_patch(os=None, select=None, socket=None, thread=None, time=None, psycopg=None)

# 如果不指定参数，默认绿化所有系统模块。
import eventlet
eventlet.monkey_patch()

# 替换部分模块
import eventlet
eventlet.monkey_patch(socket=True, select=True)
```

##### 主要的模块

###### backdoor – Python interactive interpreter within a running process

`backdoor` 模块用来可以方便的查看一个长时间运行的进程的状态。它提供了普通 Python 解释器的交互式执行的使用方式，而又不会阻塞应用程序的执行。这对于debug、性能调试等是很有用处的。

在应用程序中，以这样的方式在一个处于监听状态的socket上启动一个 `backdoor_server` ：

```python
eventlet.spawn(backdoor.backdoor_server, eventlet.listen(('localhost', 3000)))
```

运行起来后，可以通过 telnet 指定端口访问 `backdoor` ：

```python
$ telnet localhost 3000
Python 2.6.2 (r262:71600, Apr 16 2009, 09:17:39)
[GCC 4.0.1 (Apple Computer, Inc. build 5250)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> import myapp
>>> dir(myapp)
['__all__', '__doc__', '__name__', 'myfunc']
>>>
```

`backdoor` 可以与应用程序中的 yield 配合，转向指定的运行中的协程，所以就算在协程切换频繁的应用程序上使用 `backdoor`, 也可以通过 Python 解释器的命令查看程序内部的状态。

`backdoor` 主要方法和类

```python
'''Sets up an interactive console on a socket with a single connected client. 
   This does not block the caller, as it spawns a new greenlet to handle the console. 
   This is meant to be called from within an accept loop (such as backdoor_server).
'''
eventlet.backdoor.backdoor(conn_info, locals=None)

'''Blocking function that runs a backdoor server on the socket sock, 
   accepting connections and running backdoor consoles for each client that connects.
   The locals argument is a dictionary that will be included in the locals() of the interpreters. 
   It can be convenient to stick important application variables in here.
'''
eventlet.backdoor.backdoor_server(sock, locals=None)
```

###### corolocal – Coroutine local storage

```python
eventlet.corolocal.get_ident()
Returns id() of current greenlet. Useful for debugging.

class eventlet.corolocal.local
```

###### debug – Debugging tools for Eventlet

###### db_pool – DBAPI 2 database connection pooling

###### event – Cross-greenthread primitive

`class eventlet.event.Event` 是若干协程之间用来通信的事件。Event 类似 Queue，只保存一项数值作为传递的结果。但二者之间存在如下两点差异：

1. 调用 `send()` 方法时，当前协程不会马上被终止执行，只有当前线程执行完毕后，hub才会切换到其他线程。
2. `send()` 方法只能被调用一次；如果想再次send得再创建一个Event。

Event事件是协程之间交互的手段，也是 `GreenThread.wait()` 实现的基础：

```python
>>> from eventlet import event
>>> import eventlet
>>> evt = event.Event()
>>> def baz(b):
...     evt.send(b + 1)
...
>>> _ = eventlet.spawn_n(baz, 3)
>>> evt.wait()
4
```

Event类的主要方法

* ready()

如果调用 wait() 方法马上返回时返回True。用来避免等待耗时过长的任务而超时。比如，使用一个列表保持多个事件，调用 ready() 方法反复地访问这些事件直到其中一个返回True，这时我们就可以 wait() 到这个事件上了。

* send(result=None, exc=None)

根据 result 编排等待队列中的执行计划(顺序)，然后立即返回到父协程(hub)。

```python
>>> from eventlet import event
>>> import eventlet
>>> evt = event.Event()
>>> def waiter():
...     print('about to wait')
...     result = evt.wait()
...     print('waited for {0}'.format(result))
>>> _ = eventlet.spawn(waiter)
>>> eventlet.sleep(0)
about to wait
>>> evt.send('a')
>>> eventlet.sleep(0)
waited for a
```

多次 send() 同一个Event事件会报错

```python
>>> evt.send('whoops')
Traceback (most recent call last):
...
AssertionError: Trying to re-send() an already-triggered event.
```

* send_exception(*args)
与 send() 方法相同，只不过是向 waiters 发送异常

```python
>>> from eventlet import event
>>> evt = event.Event()
>>> evt.send_exception(RuntimeError())
>>> evt.wait()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "eventlet/event.py", line 120, in wait
    current.throw(*self._exc)
RuntimeError
```

* wait()
等待，直到其他协程调用 send() 方法，返回其他协程调用 send() 方法传入的参数。

```python
>>> from eventlet import event
>>> import eventlet
>>> evt = event.Event()
>>> def wait_on():
...    retval = evt.wait()
...    print("waited for {0}".format(retval))
>>> _ = eventlet.spawn(wait_on)
>>> evt.send('result')
>>> eventlet.sleep(0)
waited for result
```

当等待的事件已经发生时，马上返回

```python
>>> evt.wait()
'result'
```

###### greenpool – Green Thread Pools

`class eventlet.greenpool.GreenPool(size=1000)`

* free()
返回池中可用的协程的数目。如果数目为0或更少，下一个 spawn() 或者 spawn_n() 方法调用会将调用协程阻塞直到有一个协程变为可用。

* imap(function, *iterables)
针对可迭代对象iterables中的每一项，调用function函数进行处理。

比如，该方法处理文件中的每一行特别方便

```python
def worker(line):
    return do_something(line)
pool = GreenPool()
for result in pool.imap(worker, open("filename", 'r')):
    print(result)
```

* resize(new_size)
在任何时间点更新协程池的最大可用数目。

* running()
返回GreenPool中，当前正在执行functions的协程数目。

spawn(function, *args, **kwargs)
启动一个新的协程，以参数args和kwargs运行function。返回值是运行function的GreenThread对象，从该对象可以获取function执行的返回值。如果GreenPool中协程数目达到最大值，spawn 会被阻塞执行，直到运行中的协程有一个完成运行并释放一个协程池slot。function是可再进入的，function中可以在相同的pool中再调用spawn方法，不必担心死锁。

* spawn_n(function, *args, **kwargs)
和spawn一样，启动一个协程运行function。不同的是spawn_n没有返回值，也拿不到function执行的结果。

* starmap(function, iterable)
This is the same as itertools.starmap() and imap(function, *iterables).

* waitall()
Waits until all greenthreads in the pool are finished working.

* waiting()
Return the number of greenthreads waiting to spawn.

###### greenpool – Green Thread Pools

`class eventlet.greenpool.GreenPile(size_or_pool=1000)`

GreenPile 是对若干IO相关任务的抽象描述。一个 GreenPile 是由一个已存在的 GreenPool 对象构成。一个 GreenPool 对象可以有多个相关联的 GreenPile。

GreenPile 也可以单独创建，不与任何 GreenPool 对象有关，创建的时候传入整数值的size作为参数。

* next()
等待下一个结果，挂起当前协程直到available。当不再有结果返回时抛出异常 StopIteration。

* spawn(func, *args, **kw)
Runs func in its own green thread, with the result available by iterating over the GreenPile object.

###### [greenthread – Green Thread Implementation](http://eventlet.net/doc/modules/greenthread.html)

###### [class eventlet.greenthread.GreenThread(parent)](http://eventlet.net/doc/modules/greenthread.html)

###### [pools - Generic pools of resources](http://eventlet.net/doc/modules/pools.html)

###### [queue – Queue class](http://eventlet.net/doc/modules/queue.html)

###### [semaphore – Semaphore classes](http://eventlet.net/doc/modules/semaphore.html)

###### [timeout – Universal Timeouts](http://eventlet.net/doc/modules/timeout.html)

###### [websocket – Websocket Server](http://eventlet.net/doc/modules/websocket.html)

###### [wsgi – WSGI server](http://eventlet.net/doc/modules/wsgi.html)

###### [eventlet.green.zmq – ØMQ support](http://eventlet.net/doc/modules/zmq.html)

###### [eventlet.green.zmq – ØMQ support](http://eventlet.net/doc/modules/zmq.html#zmq-the-pyzmq-omq-python-bindings)

排版好乱，喵了个喵……
