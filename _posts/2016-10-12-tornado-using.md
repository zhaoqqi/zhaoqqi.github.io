---
layout: post
title: 使用 Tornado
categories: [Tornado]
description: Using Tornado.
keywords: Tornado
---

本来想着使用蚂蚁搬家的方式，将 Tornado 整体框架分割为各个章节，各个方面分别学习整理。但发现这样的效率不高，经常碰到需要深入或者扩展的知识点。所以决定先使用，在用的过程中摸清楚整个框架活的主线，通过主线逐渐再渗透的细节。那么，这一篇文章就记录和整理一下使用 Tornado 框架开发应用的心得。

本篇文章，我们尝试将 OpenStack 的 nova-api 服务跑在 Tornado 上，正好可以与 eventlet 做一下并发处理性能比较。

#### 初步实现 Web 服务

首先看下，我们通过官网上的第一个代码示例认识下 tornado 框架的用法：

```python
import tornado.ioloop
import tornado.web

class MainHandler(tornado.web.Requesthandler):
    def get(self):
        self.write("Hello, world!")

def make_app():
    return tornado.web.Application([
        (r"/", MainHandler),
    ])

if __name__ == '__main__':
    app = make_app()
    app.listen(8879)
    tornado.ioloop.IOLoop().current().start()
```

以上代码将启动一个 Web 服务，当访问本机的8879端口，即 `127.0.0.1:8879` 时，服务端返回 `Hello, world!`。

但是，OpenStack 的 nova-api 服务是一个 wsgi 服务。我们看下 Tornado 框架如何支持 wsgi 。

#### wsgi 服务的支持

参考官网：http://www.tornadoweb.org/en/stable/wsgi.html

##### Running WSGI apps on Tornado servers

`tornado.wsgi.WsgiContainer` 模块可以让其他 wsgi 应用或者框架在 Tornado HTTP 服务上运行。For example, with this class you can mix Django and Tornado handlers in a single server.

还是先看下官网的例子：

```python
def simple_app(environ, start_response):
    status = "200 OK"
    response_headers = [("Content-type", "text/plain")]
    start_response(status, response_headers)
    return ["Hello world!\n"]

container = tornado.wsgi.WSGIContainer(simple_app)
http_server = tornado.httpserver.HTTPServer(container)
http_server.listen(8888)
tornado.ioloop.IOLoop.current().start()
```

参考上面的例子程序，在 Tornado 上运行 nova-api 服务的思路为，找到 nova-api 的 wsgi Application，使用 `WSGIContainer` 包装为 http_server，启动 tornado 的 ioloop 即可。

明天继续……
