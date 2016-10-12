---
layout: post
title: Eventlet(4) - nova-api如何使用Eventlet
categories: [OpenStack]
description: How nova-api use eventlet.
keywords: OpenStack, Python, nova-api, Eventlet
---

#### 服务启动流程分析

本文通过 nova-api 服务启动的代码执行流程，具体了解一下 nova-api 服务是如何使用 Eventlet 框架的。

首先是 nova-api 的启动脚本

```python
if __name__ == '__main__':
    flags.parse_args(sys.argv)
    logging.setup("nova")
    utils.monkey_patch()

    # 初始化一个 ProcessLauncher 实例
    launcher = service.ProcessLauncher()
    for api in flags.FLAGS.enabled_apis:
        # 初始化 wsgi service，server名称为 'osapi_compute'
        server = service.WSGIService(api)
        # 启动 wsgi service中的 server
        launcher.launch_server(server, workers=server.workers or 1)
    launcher.wait()
```

ProcessLauncher类和 WSGIService类的初始化方法如下。前者用来启动并管理 wsgi服务，后者用来从 api-paste.ini 的配置中启动 nova-api 服务。

```python
class ProcessLauncher(object):
    def __init__(self):
        self.children = {}
        self.sigcaught = None
        self.running = True
        rfd, self.writepipe = os.pipe()
        self.readpipe = eventlet.greenio.GreenPipe(rfd, 'r')

        signal.signal(signal.SIGTERM, self._handle_signal)
        signal.signal(signal.SIGINT, self._handle_signal)

    def launch_server(self, server, workers=1):
        wrap = ServerWrapper(server, workers)

        LOG.info(_('Starting %d workers'), wrap.workers)
        while self.running and len(wrap.children) < wrap.workers:
            self._start_child(wrap)


class WSGIService(object):
    """Provides ability to launch API from a 'paste' configuration."""

    def __init__(self, name, loader=None):
        """Initialize, but do not start the WSGI server.

        :param name: The name of the WSGI server given to the loader.
        :param loader: Loads the WSGI application using the given name.
        :returns: None

        """
        self.name = name
        self.manager = self._get_manager()
        self.loader = loader or wsgi.Loader()
        self.app = self.loader.load_app(name)
        self.host = getattr(FLAGS, '%s_listen' % name, "0.0.0.0")
        self.port = getattr(FLAGS, '%s_listen_port' % name, 0)
        self.workers = getattr(FLAGS, '%s_workers' % name, None)
        self.server = wsgi.Server(name,
                                  self.app,
                                  host=self.host,
                                  port=self.port)
        # Pull back actual port used
        self.port = self.server.port
```

`launcher.launch_server(server, workers=server.workers or 1)` 真正开始启动 nova-api 服务线程。`self._start_child()` 方法启动一个子进程，子进程调用 `self._child_process()` 方法，实例化一个 Launcher 对象来启动真正的 nova-api server服务。

```python
class ProcessLauncher(object):
    
    ......

    def launch_server(self, server, workers=1):
        wrap = ServerWrapper(server, workers)

        LOG.info(_('Starting %d workers'), wrap.workers)
        while self.running and len(wrap.children) < wrap.workers:
            self._start_child(wrap)

    def _start_child(self, wrap):
        if len(wrap.forktimes) > wrap.workers:
            # Limit ourselves to one process a second (over the period of
            # number of workers * 1 second). This will allow workers to
            # start up quickly but ensure we don't fork off children that
            # die instantly too quickly.
            if time.time() - wrap.forktimes[0] < wrap.workers:
                LOG.info(_('Forking too fast, sleeping'))
                time.sleep(1)

            wrap.forktimes.pop(0)

        wrap.forktimes.append(time.time())

        pid = os.fork()
        if pid == 0:
            # NOTE(johannes): All exceptions are caught to ensure this
            # doesn't fallback into the loop spawning children. It would
            # be bad for a child to spawn more children.
            status = 0
            try:
                self._child_process(wrap.server)
            except SignalExit as exc:
                signame = {signal.SIGTERM: 'SIGTERM',
                           signal.SIGINT: 'SIGINT'}[exc.signo]
                LOG.info(_('Caught %s, exiting'), signame)
                status = exc.code
            except SystemExit as exc:
                status = exc.code
            except BaseException:
                LOG.exception(_('Unhandled exception'))
                status = 2
            finally:
                wrap.server.stop()

            os._exit(status)

        LOG.info(_('Started child %d'), pid)

        wrap.children.add(pid)
        self.children[pid] = wrap

        return pid

    def _child_process(self, server):
        # Setup child signal handlers differently
        def _sigterm(*args):
            signal.signal(signal.SIGTERM, signal.SIG_DFL)
            raise SignalExit(signal.SIGTERM)

        signal.signal(signal.SIGTERM, _sigterm)
        # Block SIGINT and let the parent send us a SIGTERM
        signal.signal(signal.SIGINT, signal.SIG_IGN)

        # Reopen the eventlet hub to make sure we don't share an epoll
        # fd with parent and/or siblings, which would be bad
        eventlet.hubs.use_hub()

        # Close write to ensure only parent has it open
        os.close(self.writepipe)
        # Create greenthread to watch for parent to close pipe
        eventlet.spawn(self._pipe_watcher)

        # Reseed random number generator
        random.seed()

        launcher = Launcher()
        launcher.run_server(server)


class Launcher(object):
    """Launch one or more services and wait for them to complete."""

    def __init__(self):
        """Initialize the service launcher.

        :returns: None

        """
        self._services = []
        eventlet_backdoor.initialize_if_enabled()

    @staticmethod
    def run_server(server):
        """Start and wait for a server to finish.

        :param service: Server to run and wait for.
        :returns: None

        """
        server.start()
        server.wait()

```

Launcher 的 `run_server()` 方法中的 `server.start()` 中的 server 为 nova.wsgi.Server。

```python
class Server(object):
    """Server class to manage a WSGI server, serving a WSGI application."""

    default_pool_size = 1000

    """init方法初始化wgsi application名称，socket监听的host和port，协议，并初始化
    一个socket服务 _socket。
    """
    def __init__(self, name, app, host='0.0.0.0', port=0, pool_size=None,
                       protocol=eventlet.wsgi.HttpProtocol, backlog=128):
        """Initialize, but do not start, a WSGI server.

        :param name: Pretty name for logging.
        :param app: The WSGI application to serve.
        :param host: IP address to serve the application.
        :param port: Port number to server the application.
        :param pool_size: Maximum number of eventlets to spawn concurrently.
        :param backlog: Maximum number of queued connections.
        :returns: None
        :raises: nova.exception.InvalidInput
        """
        self.name = name
        self.app = app
        self._server = None
        self._protocol = protocol
        self._pool = eventlet.GreenPool(pool_size or self.default_pool_size)
        self._logger = logging.getLogger("nova.%s.wsgi.server" % self.name)
        self._wsgi_logger = logging.WritableLogger(self._logger)

        if backlog < 1:
            raise exception.InvalidInput(
                    reason='The backlog must be more than 1')

        self._socket = eventlet.listen((host, port), backlog=backlog)
        (self.host, self.port) = self._socket.getsockname()
        LOG.info(_("%(name)s listening on %(host)s:%(port)s") % self.__dict__)

    """调用eventlet的spawn方法，启动一个绿色线程来处理连接到_socket的http请求。
    """
    def start(self):
        """Start serving a WSGI application.

        :returns: None
        """
        self._server = eventlet.spawn(eventlet.wsgi.server,
                                      self._socket,
                                      self.app,
                                      protocol=self._protocol,
                                      custom_pool=self._pool,
                                      log=self._wsgi_logger)
```

接下来我们看下 `eventlet.spawn()` 方法。

```python
# eventlet/greenthread.py

def spawn(func, *args, **kwargs):
    """Create a greenthread to run ``func(*args, **kwargs)``.  Returns a
    :class:`GreenThread` object which you can use to get the results of the
    call.

    Execution control returns immediately to the caller; the created greenthread
    is merely scheduled to be run at the next available opportunity.
    Use :func:`spawn_after` to  arrange for greenthreads to be spawned
    after a finite delay.
    """
    hub = hubs.get_hub()
    g = GreenThread(hub.greenlet)
    hub.schedule_call_global(0, g.switch, func, args, kwargs)
    return g
```

首先获取全局单例对象 hub，以 hub.greenlet 为父亲线程创建一个绿色线程 g。将绿色线程 g 放入 hub 的待调度执行队列中。方法最后返回绿色线程对象 g。

再看下绿色线程 g(eventlet.wsgi.server()方法) 的内容：使用传入参数初始化一个 Server 对象，以及一个 GreenPool 对象。传入的sock阻塞accept循环等待客户端连接，每次有客户端连接，绿色线程池都分配一个绿色线程处理这个连接的请求。该绿色线程传入的执行方法为 `pool.spawn_n(serv.process_request, client_socket)` 。

```python
# eventlet/wsgi.py

def server(sock, site,
           log=None,
           environ=None,
           max_size=None,
           max_http_version=DEFAULT_MAX_HTTP_VERSION,
           protocol=HttpProtocol,
           server_event=None,
           minimum_chunk_size=None,
           log_x_forwarded_for=True,
           custom_pool=None,
           keepalive=True,
           log_output=True,
           log_format=DEFAULT_LOG_FORMAT,
           url_length_limit=MAX_REQUEST_LINE,
           debug=True,
           socket_timeout=None,
           capitalize_response_headers=True):
    """Start up a WSGI server handling requests from the supplied server
    socket.  This function loops forever.  The *sock* object will be
    closed after server exits, but the underlying file descriptor will
    remain open, so if you have a dup() of *sock*, it will remain usable.

    .. warning::

        At the moment :func:`server` will always wait for active connections to finish before
        exiting, even if there's an exception raised inside it
        (*all* exceptions are handled the same way, including :class:`greenlet.GreenletExit`
        and those inheriting from `BaseException`).

        While this may not be an issue normally, when it comes to long running HTTP connections
        (like :mod:`eventlet.websocket`) it will become problematic and calling
        :meth:`~eventlet.greenthread.GreenThread.wait` on a thread that runs the server may hang,
        even after using :meth:`~eventlet.greenthread.GreenThread.kill`, as long
        as there are active connections.

    :param sock: Server socket, must be already bound to a port and listening.
    :param site: WSGI application function.
    :param log: logging.Logger instance or file-like object that logs should be written to.
                If a Logger instance is supplied, messages are sent to the INFO log level.
                If not specified, sys.stderr is used.
    :param environ: Additional parameters that go into the environ dictionary of every request.
    :param max_size: Maximum number of client connections opened at any time by this server.
                Default is 1024.
    :param max_http_version: Set to "HTTP/1.0" to make the server pretend it only supports HTTP 1.0.
                This can help with applications or clients that don't behave properly using HTTP 1.1.
    :param protocol: Protocol class.  Deprecated.
    :param server_event: Used to collect the Server object.  Deprecated.
    :param minimum_chunk_size: Minimum size in bytes for http chunks.  This can be used to improve
                performance of applications which yield many small strings, though
                using it technically violates the WSGI spec. This can be overridden
                on a per request basis by setting environ['eventlet.minimum_write_chunk_size'].
    :param log_x_forwarded_for: If True (the default), logs the contents of the x-forwarded-for
                header in addition to the actual client ip address in the 'client_ip' field of the
                log line.
    :param custom_pool: A custom GreenPool instance which is used to spawn client green threads.
                If this is supplied, max_size is ignored.
    :param keepalive: If set to False, disables keepalives on the server; all connections will be
                closed after serving one request.
    :param log_output: A Boolean indicating if the server will log data or not.
    :param log_format: A python format string that is used as the template to generate log lines.
                The following values can be formatted into it: client_ip, date_time, request_line,
                status_code, body_length, wall_seconds.  The default is a good example of how to
                use it.
    :param url_length_limit: A maximum allowed length of the request url. If exceeded, 414 error
                is returned.
    :param debug: True if the server should send exception tracebacks to the clients on 500 errors.
                If False, the server will respond with empty bodies.
    :param socket_timeout: Timeout for client connections' socket operations. Default None means
                wait forever.
    :param capitalize_response_headers: Normalize response headers' names to Foo-Bar.
                Default is True.
    """
    serv = Server(sock, sock.getsockname(),
                  site, log,
                  environ=environ,
                  max_http_version=max_http_version,
                  protocol=protocol,
                  minimum_chunk_size=minimum_chunk_size,
                  log_x_forwarded_for=log_x_forwarded_for,
                  keepalive=keepalive,
                  log_output=log_output,
                  log_format=log_format,
                  url_length_limit=url_length_limit,
                  debug=debug,
                  socket_timeout=socket_timeout,
                  capitalize_response_headers=capitalize_response_headers,
                  )
    if server_event is not None:
        server_event.send(serv)
    if max_size is None:
        max_size = DEFAULT_MAX_SIMULTANEOUS_REQUESTS
    if custom_pool is not None:
        pool = custom_pool
    else:
        pool = greenpool.GreenPool(max_size)
    try:
        serv.log.info("(%s) wsgi starting up on %s" % (
            serv.pid, socket_repr(sock)))
        while is_accepting:
            try:
                client_socket = sock.accept()
                client_socket[0].settimeout(serv.socket_timeout)
                serv.log.debug("(%s) accepted %r" % (
                    serv.pid, client_socket[1]))
                try:
                    pool.spawn_n(serv.process_request, client_socket)
                except AttributeError:
                    warnings.warn("wsgi's pool should be an instance of "
                                  "eventlet.greenpool.GreenPool, is %s. Please convert your"
                                  " call site to use GreenPool instead" % type(pool),
                                  DeprecationWarning, stacklevel=2)
                    pool.execute_async(serv.process_request, client_socket)
            except ACCEPT_EXCEPTIONS as e:
                if support.get_errno(e) not in ACCEPT_ERRNO:
                    raise
            except (KeyboardInterrupt, SystemExit):
                serv.log.info("wsgi exiting")
                break
    finally:
        pool.waitall()
        serv.log.info("(%s) wsgi exited, is_accepting=%s" % (
            serv.pid, is_accepting))
        try:
            # NOTE: It's not clear whether we want this to leave the
            # socket open or close it.  Use cases like Spawning want
            # the underlying fd to remain open, but if we're going
            # that far we might as well not bother closing sock at
            # all.
            sock.close()
        except socket.error as e:
            if support.get_errno(e) not in BROKEN_SOCK:
                traceback.print_exc()


class Server(BaseHTTPServer.HTTPServer):

    def __init__(self,
                 socket,
                 address,
                 app,
                 log=None,
                 environ=None,
                 max_http_version=None,
                 protocol=HttpProtocol,
                 minimum_chunk_size=None,
                 log_x_forwarded_for=True,
                 keepalive=True,
                 log_output=True,
                 log_format=DEFAULT_LOG_FORMAT,
                 url_length_limit=MAX_REQUEST_LINE,
                 debug=True,
                 socket_timeout=None,
                 capitalize_response_headers=True):

        self.outstanding_requests = 0
        self.socket = socket
        self.address = address
        if log:
            self.log = get_logger(log, debug)
        else:
            self.log = get_logger(sys.stderr, debug)
        self.app = app
        self.keepalive = keepalive
        self.environ = environ
        self.max_http_version = max_http_version
        self.protocol = protocol
        self.pid = os.getpid()
        self.minimum_chunk_size = minimum_chunk_size
        self.log_x_forwarded_for = log_x_forwarded_for
        self.log_output = log_output
        self.log_format = log_format
        self.url_length_limit = url_length_limit
        self.debug = debug
        self.socket_timeout = socket_timeout
        self.capitalize_response_headers = capitalize_response_headers

        if not self.capitalize_response_headers:
            warnings.warn("""capitalize_response_headers is disabled.
 Please, make sure you know what you are doing.
 HTTP headers names are case-insensitive per RFC standard.
 Most likely, you need to fix HTTP parsing in your client software.""",
                          DeprecationWarning, stacklevel=3)

    def get_environ(self):
        d = {
            'wsgi.errors': sys.stderr,
            'wsgi.version': (1, 0),
            'wsgi.multithread': True,
            'wsgi.multiprocess': False,
            'wsgi.run_once': False,
            'wsgi.url_scheme': 'http',
        }
        # detect secure socket
        if hasattr(self.socket, 'do_handshake'):
            d['wsgi.url_scheme'] = 'https'
            d['HTTPS'] = 'on'
        if self.environ is not None:
            d.update(self.environ)
        return d

    def process_request(self, sock_params):
        # The actual request handling takes place in __init__, so we need to
        # set minimum_chunk_size before __init__ executes and we don't want to modify
        # class variable
        sock, address = sock_params
        proto = new(self.protocol)
        if self.minimum_chunk_size is not None:
            proto.minimum_chunk_size = self.minimum_chunk_size
        proto.capitalize_response_headers = self.capitalize_response_headers
        try:
            proto.__init__(sock, address, self)
        except socket.timeout:
            # Expected exceptions are not exceptional
            sock.close()
            # similar to logging "accepted" in server()
            self.log.debug('(%s) timed out %r' % (self.pid, address))

    def log_message(self, message):
        warnings.warn('server.log_message is deprecated.  Please use server.log.info instead')
        self.log.info(message)

```

每当一个绿色线程的任务执行完成，或者因为IO操作被挂起，绿色线程的执行权就被 switch 到绿色线程池 pool 中所有绿色线程的父亲线程 `g.greenlet` 来执行。`g.greenlet` 即是 hub 对象初始化时指定的 `greenlet.greenlet(self.run)`，即是Hub对象中的 `self.run()` 方法。该方法作为调度中心，指定下一个该由哪个绿色线程开始执行。

```python
# eventlet/hubs/hub.py

class BaseHub(object):
    """ Base hub class for easing the implementation of subclasses that are
    specific to a particular underlying event architecture. """

    def run(self, *a, **kw):
            """Run the runloop until abort is called.
            """
            # accept and discard variable arguments because they will be
            # supplied if other greenlets have run and exited before the
            # hub's greenlet gets a chance to run
            if self.running:
                raise RuntimeError("Already running!")
            try:
                self.running = True
                self.stopping = False
                while not self.stopping:
                    while self.closed:
                        # We ditch all of these first.
                        self.close_one()
                    self.prepare_timers()
                    if self.debug_blocking:
                        self.block_detect_pre()
                    self.fire_timers(self.clock())
                    if self.debug_blocking:
                        self.block_detect_post()
                    self.prepare_timers()
                    wakeup_when = self.sleep_until()
                    if wakeup_when is None:
                        sleep_time = self.default_sleep()
                    else:
                        sleep_time = wakeup_when - self.clock()
                    if sleep_time > 0:
                        self.wait(sleep_time)
                    else:
                        self.wait(0)
                else:
                    self.timers_canceled = 0
                    del self.timers[:]
                    del self.next_timers[:]
            finally:
                self.running = False
                self.stopping = False
```

#### 小结

nova-api 启动服务时，会根据 nova.conf 中配置的 workers 数量启动指定数量的子进程。每个子进程初始化一个以 hub 对象的self.greenlet为父亲线程的绿色线程池，以及一个监听 nova-api服务器和指定端口号的 socket 服务。每当有客户端请求该socket服务，绿色线程池都会生成一个绿色线程处理该请求。当有多个请求到达时，同一时间只有一个绿色线程处于工作状态，其他绿色线程等待。当工作的绿色线程完成工作或者被IO任务阻塞时，线程的执行被切换到绿色线程池中所有绿色线程的父亲线程，即 hub 对象的self.greenlet执行 (即hub对象的self.run()方法)。父亲线程会根据其他绿色线程的就绪状态，选择出接下来可以执行的绿色线程，并切换到该绿色线程执行。