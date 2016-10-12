---
layout: post
title: Tornado Intro
categories: [Tornado]
description: Introduction of Tornado.
keywords: Tornado
---

以下内容翻译自 [Tornador官方文档](http://www.tornadoweb.org/en/stable)

#### 介绍

Tornado 是一个 Python Web 框架，一个异步的网络编程库，最初由 `FriendFeed` 开发。Tornado 通过使用异步的非阻塞式的网络IO，可以同时支持成千上万的请求连接，这对于长时间轮询，WebSocket 服务，以及其他长连接应用来说是特别理想的。

Tornado 可以大致分为以下几个主要组成部分：
* 一个 Web 框架(包含用来创建 web application 的子类 `RequestHandler` 在内的一系列类)
* HTTP 协议的服务器端和客户端的具体实现(`HTTPServer` 和 `AsyncHTTPClient`)
* 一个异步网络编程库，主要包含类 `IOLoop` 和 `IOStream` ，作为 HTTP 组件的构建块，也可以作为其他协议组件的构建块。
* 一个协程库(`tornado.gen`)，其允许使用更直接的方式实现异步代码，而不是使用链式回调的方式。

Tornado web 框架和HTTP服务器一起，提供了一个完整的 WSGI 的替代品。虽然也可以将 Tornado web 框架单独在一个 WSGI 容器中使用，也可以将 Tornado HTTP 服务器作为一个容器在其他 WSGI 框架中使用，但是单独每个个体组件不能发挥 Tornado 的所有优势，只有两者组合使用才可以发挥 Tornado 框架的最大优势。


#### 异步和非阻塞式IO

实时响应的 web 功能需要为每一个用户提供一个长时连接。对于传统的同步web服务来说，意味着要为每个用户提供一个线程，这个开销相对昂贵。

为了减小大批量并发连接的开销，Tornado 使用一个单例线程化的事件循环(event loop)来处理。这就意味着所有的应用程序必须以异步和非阻塞式为目标开发，因为在事件循环中，在同一时刻只能处理一个应用程序的请求。

异步和非阻塞这两个术语密切相关，在一些情况下甚至可以互相替换使用，但它们是不一样的。

##### 阻塞

一个方法(函数)被阻塞是指，在返回之前因为某个事件未发生而一直处于等待状态。多个原因会导致一个方法(函数)被阻塞：网络I/O，磁盘I/O，互斥锁等。实际上，当某个方法(函数)在运行和使用CPU时，多多少少也会阻塞(为了说明CPU阻塞和其他请求导致的阻塞一样需要重视，我们举一个略极端的例子。想想密码散列方法比如`bcrypt`，运行时会消耗几百微妙的CPU时间，这个时间远远大于一个典型的磁盘或者网络I/O访问)。

一个方法(函数)在某些方面会被阻塞，在其他一些方面则不会。举例来说，`tornado.httpclient`使用默认配置时会阻塞在DNS解析时，但在其他网络访问时则不会()。在 Tornado 的语言环境中，我们经常讨论的阻塞是网络I/O。

##### 异步

一个异步的方法(函数)在执行结束之前就返回了，通常在应用程序触发一些未来的动作之前，使一些动作在后台发生(同步方法/函数与之相反，都是在返回之前将所有工作完成)。有许多类型的异步编程接口，主要有以下四种：
* 回调参数 - Callback argument
* 返回一个占位符 - Return a placeholder (`Future`, `Promise`, `Deferred`)
* 传递给一个队列 - Deliver to a queue
* 回调注册表 - Callback registry (e.g. POSIX signals)

无论使用哪种类型的异步编程接口，异步方法(函数)显然与它们的调用者有不同的交互作用。没有免费的方法让一个同步方法(函数)变为异步，对于它们的调用者来说也是不透明的(gevent使用轻量级的线程提供了性能上可以与异步系统相比较的能力，但是它并没有真正做到异步处理请求)。

看一个同步方法的例子：

```python
from tornado.httpclient import HTTPClient

def synchronous_fetch(url):
    http_client = HTTPClient()
    response = http_client.fetch(url)
    return response.body
```

接着看一个使用回调参数实现的相同功能的异步方法：

```python
from tornado.httpclient import AsyncHTTPClient

def asynchronous_fetch(url, callback):
    http_client = AsyncHTTPClient()
    def handle_response(response):
        callback(response.body)
    http_client.fetch(url, callback=handle_response)
```

再看一个使用`Future`实现的异步方法：

```python
from tornado.concurrent import Future
from tornado.httpclient import AsyncHTTPClient

def async_fetch_future(url):
    http_client = AsyncHTTPClient()
    my_future = Future()
    fetch_future = http_client.fetch(url)
    fetch_future.add_done_callback(
        lambda f: my_future.set_result(f.result()))
    return my_future
```

原生版本的`Future`会更复杂，尽管如此，`Futures`在 Tornado 中还是推荐的方案，因为它的两个主要优势。因为`Future.result `能够简单的抛出一个异常，使它有更加一致的错误处理能力(不像回调机制编程接口那样需要专门的错误处理)；同时，`Futures`可以很好的使用协程。关于协程会在下一节做更深入的讨论。现在我们只是看下使用协程如何实现我们的例子程序，看起来有点像使用同步方法的版本：

```python
from tornado import gen

@gen.coroutine
def fetch_coroutine(url):
    http_client = AsyncHTTPClient()
    response = yield http_client.fetch(url)
    raise gen.Return(response.body)
```

语句 `raise gen.Return(response.body)` 是为python2.x特别实现的，因为python2.x的生成器不允许返回值。为了达到这个目的，Tornado的协程抛出一种特别类型的异常 `Return`。协程会捕捉到这个异常，并把它当作一个返回值看待。在python3.3及以后，使用`return response.body`达到相同的结果。


#### 协程 - Coroutines

在 Tornado 中，使用协程是被推荐的实现异步代码的方式。协程使用 Python关键字`yield`实现代码执行的挂起运行(suspend)和恢复运行(resume)，从而替换掉链式回调的方式(在像`gevent`这样的系统中，可以相互协作的轻量级的线程也被称为协程，但在 Tornado 中，所有的协程都使用显示的上下文切换执行，被称为异步方法 - 原文为'and are called as asynchronous functions')。

协程不仅没有额外的一个线程的开销，而且和同步的编码一样简单。

##### 协程怎样工作

包含`yield`的方法(函数)就是一个生成器。所有的生成器都是异步的，每次被调用时返回一个生成器对象而不是运行直到方法结束。`@gen.coroutine`装饰器通过`yield`关键字与生成器交互，通过返回一个`Future`与协程的调用者交互。

下面是一个简化版本的协程装饰器的内部循环：

```python
# Simplified inner loop of tornado.gen.Runner
def run(self):
    # send(x) makes the current yield return x.
    # It returns when the next yield is reached
    future = self.gen.send(self.next)
    def callback(f):
        self.next = f.result()
        self.run()
    future.add_done_callback(callback)
```

装饰器从生成器接收一个`Future`对象，等待(非阻塞)`Future`对象执行完成。然后"解包"`future`对象，发送处理结果返回给生成器当作`yield`表达式的结果。绝大多数的异步代码都不直接使用类`Future`，除非将`Future`对象通过异步方法(函数)立即返回给`yield`表达式。


##### 如何调用一个协程

协程不使用正常的方式抛出异常：任何一个异常在抛出之前都陷入在(trapped in)一个 `Futrue` 对象里，直到它 `yield`。这意味着使用正确的方式调用协程很重要，否则你可能会遇到未被显著察觉的错误：

```python
@gen.coroutine
def divide(x, y):
    return x / y

def bad_call():
    # This should raise a ZeroDivisionError, but it won't because
    # the coroutine is called incorrectly.
    divide(1, 0)
```

在几乎所有的使用场景中，调用协程的任何一个方法(函数)其本身必须首先是一个协程，而且在调用协程时要使用关键字 `yield`。当你重写一个父类定义的方法时，要查阅说明文档以确定是否支持协程(文档会说明此方法"may be a coroutine"或者"may return a `Future`")：

```python
@gen.coroutine
def good_call():
    # yield will unwrap the Future returned by divide() and raise
    # the exception.
    yield divide(1, 0)
```

有时你会想生成一个协程执行任务，并且不必等到它的执行结果。在这个场景下，推荐使用 `IOLoop.spawn_callback`，这会使 `IOLoop` 变的可靠。如果执行失败，`IOLoop` 会在日志中记录下错误信息的stack trace：

```python
# The IOLoop will catch the exception and print a stack trace in
# the logs. Note that this doesn't look like a normal call, since
# we pass the function object to be called by the IOLoop.
IOLoop.current().spawn_callback(divide, 1, 0)
```

本想把 Tornado 官方文档的 User Guide 翻译完，当作简介性的文字放在 Tornado 系列文章的第一篇。但……太TM长了……而且很多相关内容还不熟练。先把 Tornado 的架构和几个重点模块学习一下，再继续写这篇……

