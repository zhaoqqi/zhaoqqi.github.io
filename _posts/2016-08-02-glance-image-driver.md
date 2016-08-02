---
layout: post
title: glance-api 服务启动流程(官方Kilo版本)
categories: [OpenStack]
description: 
keywords: Glance
---

glance-api 启动入口在 glance/cmd/api.py

```python
def main():
    try:
        config.parse_args()
        wsgi.set_eventlet_hub()
        logging.setup(CONF, 'glance')

        if cfg.CONF.profiler.enabled:
            _notifier = osprofiler.notifier.create("Messaging",
                                                   notifier.messaging, {},
                                                   notifier.get_transport(),
                                                   "glance", "api",
                                                   cfg.CONF.bind_host)
            osprofiler.notifier.set(_notifier)
        else:
            osprofiler.web.disable()

        server = wsgi.Server(initialize_glance_store=True)
        server.start(config.load_paste_app('glance-api'), default_port=9292)
        server.wait()
    except KNOWN_EXCEPTIONS as e:
        fail(e)

if __name__ == '__main__':
    main()
```

设置 disable_process_locking 和 lock_path 的默认值：

```python
def parse_args(args=None, usage=None, default_config_files=None):
    if "OSLO_LOCK_PATH" not in os.environ:
        lockutils.set_defaults(tempfile.gettempdir())

    # 设置 project='glance', version='2015.1.0'
    CONF(args=args,
         project='glance',
         version=version.cached_version_string(),
         usage=usage,
         default_config_files=default_config_files)

def set_defaults(lock_path):
    """Set value for lock_path.

    This can be used by tests to set lock_path to a temporary directory.
    """

    # lock_path='/tmp'
    # _opts = [<oslo_config.cfg.BoolOpt object at 0x2d14b50>, <oslo_config.cfg.StrOpt object at 0x2d14c10>]
    # _opts[0].name='disable_process_locking', _opts[1].name='lock_path'
    cfg.set_defaults(_opts, lock_path=lock_path)
```

wsgi.set_eventlet_hub() 设置协程调度中心使用'epoll'，如果不支持'epoll'则使用'selects'：

```python
def set_eventlet_hub():
    try:
        eventlet.hubs.use_hub('poll')
    except Exception:
        try:
            eventlet.hubs.use_hub('selects')
        except Exception:
            msg = _("eventlet 'poll' nor 'selects' hubs are available "
                    "on this platform")
            raise exception.WorkerCreationFailure(
                reason=msg)
```

接下来这段使用osprofiler模块，可以配合Ceilometer实现性能分析，暂时忽略……


server = wsgi.Server(initialize_glance_store=True)
初始化wsgi server：

```python
def __init__(self, threads=1000, initialize_glance_store=False):
    os.umask(0o27)  # ensure files are created with the correct privileges
    self._logger = logging.getLogger("eventlet.wsgi.server")
    self._wsgi_logger = loggers.WritableLogger(self._logger)
    self.threads = threads
    self.children = set()
    self.stale_children = set()
    self.running = True
    # NOTE(abhishek): Allows us to only re-initialize glance_store when
    # the API's configuration reloads.
    self.initialize_glance_store = initialize_glance_store
    self.pgid = os.getpid()
    try:
        # NOTE(flaper87): Make sure this process
        # runs in its own process group.
        os.setpgid(self.pgid, self.pgid)
    except OSError:
        # NOTE(flaper87): When running glance-control,
        # (glance's functional tests, for example)
        # setpgid fails with EPERM as glance-control
        # creates a fresh session, of which the newly
        # launched service becomes the leader (session
        # leaders may not change process groups)
        #
        # Running glance-(api|registry) is safe and
        # shouldn't raise any error here.
        self.pgid = 0
```

启动 wsgi server前 load_paste_app:

```python
def load_paste_app(app_name, flavor=None, conf_file=None):
    """
    Builds and returns a WSGI app from a paste config file.

    We assume the last config file specified in the supplied ConfigOpts
    object is the paste config file, if conf_file is None.

    :param app_name: name of the application to load
    :param flavor: name of the variant of the application to load
    :param conf_file: path to the paste config file

    :raises RuntimeError when config file cannot be located or application
            cannot be loaded from config file
    """
    # append the deployment flavor to the application name,
    # in order to identify the appropriate paste pipeline
    app_name += _get_deployment_flavor(flavor) //app_name = 'glance-api'

    if not conf_file:
        conf_file = _get_deployment_config_file()

    try:
        logger = logging.getLogger(__name__)
        logger.debug("Loading %(app_name)s from %(conf_file)s",
                     {'conf_file': conf_file, 'app_name': app_name})

        app = deploy.loadapp("config:%s" % conf_file, name=app_name)

        # Log the options used when starting if we're in debug mode...
        if CONF.debug:
            CONF.log_opt_values(logger, logging.DEBUG)

        # app 为<glance.api.middleware.version_negotiation.VersionNegotiationFilter>对象
        return app
    except (LookupError, ImportError) as e:
        msg = (_("Unable to load %(app_name)s from "
                 "configuration file %(conf_file)s."
                 "\nGot: %(e)r") % {'app_name': app_name,
                                    'conf_file': conf_file,
                                    'e': e})
        logger.error(msg)
        raise RuntimeError(msg)
```

启动 wsgi server：

```python
def start(self, application, default_port):
    """
    Run a WSGI server with the given application.

    :param application: The application to be run in the WSGI server
    :param default_port: Port to bind to if none is specified in conf
    """
    self.application = application
    self.default_port = default_port
    self.configure()
    self.start_wsgi()

def configure(self, old_conf=None, has_changed=None):
    """
    Apply configuration settings

    :param old_conf: Cached old configuration settings (if any)
    :param has changed: callable to determine if a parameter has changed
    """
    eventlet.wsgi.MAX_HEADER_LINE = CONF.max_header_line //16384
    self.configure_socket(old_conf, has_changed)
    if self.initialize_glance_store:
        initialize_glance_store()

    def configure_socket(self, old_conf=None, has_changed=None):
        """
        Ensure a socket exists and is appropriately configured.

        This function is called on start up, and can also be
        called in the event of a configuration reload.

        When called for the first time a new socket is created.
        If reloading and either bind_host or bind port have been
        changed the existing socket must be closed and a new
        socket opened (laws of physics).

        In all other cases (bind_host/bind_port have not changed)
        the existing socket is reused.

        :param old_conf: Cached old configuration settings (if any)
        :param has changed: callable to determine if a parameter has changed
        """
        # Do we need a fresh socket?
        new_sock = (old_conf is None or (
                    has_changed('bind_host') or
                    has_changed('bind_port')))
        # Will we be using https?
        use_ssl = not (not CONF.cert_file or not CONF.key_file)
        # Were we using https before?
        old_use_ssl = (old_conf is not None and not (
                       not old_conf.get('key_file') or
                       not old_conf.get('cert_file')))
        # Do we now need to perform an SSL wrap on the socket?
        wrap_sock = use_ssl is True and (old_use_ssl is False or new_sock)
        # Do we now need to perform an SSL unwrap on the socket?
        unwrap_sock = use_ssl is False and old_use_ssl is True

        if new_sock:
            self._sock = None
            if old_conf is not None:
                self.sock.close()
            _sock = get_socket(self.default_port)
            _sock.setsockopt(socket.SOL_SOCKET,
                             socket.SO_REUSEADDR, 1)
            # sockets can hang around forever without keepalive
            _sock.setsockopt(socket.SOL_SOCKET,
                             socket.SO_KEEPALIVE, 1)
            self._sock = _sock

        if wrap_sock:
            self.sock = ssl_wrap_socket(self._sock)

        if unwrap_sock:
            self.sock = self._sock

        if new_sock and not use_ssl:
            self.sock = self._sock

        # Pick up newly deployed certs
        if old_conf is not None and use_ssl is True and old_use_ssl is True:
            if has_changed('cert_file') or has_changed('key_file'):
                utils.validate_key_cert(CONF.key_file, CONF.cert_file)
            if has_changed('cert_file'):
                self.sock.certfile = CONF.cert_file
            if has_changed('key_file'):
                self.sock.keyfile = CONF.key_file

        if new_sock or (old_conf is not None and has_changed('tcp_keepidle')):
            # This option isn't available in the OS X version of eventlet
            if hasattr(socket, 'TCP_KEEPIDLE'):
                self.sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPIDLE,
                                     CONF.tcp_keepidle)

        if old_conf is not None and has_changed('backlog'):
            self.sock.listen(CONF.backlog)

"""根据 /etc/glance/glance-api.conf 中的[glance_store]
   加载后端存储 driver
"""
def initialize_glance_store():
    """Initialize glance store."""
    glance_store.register_opts(CONF)
    glance_store.create_stores(CONF)
    glance_store.verify_default_store()

"""根据 /etc/glance/glance-api.conf 中的[workers]
   启动指定数量的服务协程
"""
def start_wsgi(self):
    if CONF.workers == 0:
        # Useful for profiling, test, debug etc.
        self.pool = self.create_pool()
        self.pool.spawn_n(self._single_run, self.application, self.sock)
        return
    else:
        LOG.info(_LI("Starting %d workers") % CONF.workers)
        signal.signal(signal.SIGTERM, self.kill_children)
        signal.signal(signal.SIGINT, self.kill_children)
        signal.signal(signal.SIGHUP, self.hup)
        while len(self.children) < CONF.workers:
            self.run_child()

"""父进程wait子进程状态。
   如果父进程 self.running==True，检测到子进程退出则重新启动子进程;
   如果父进程 self.running==False，检测到子退出则 shutdown_safe;
"""
def wait(self):
    """Wait until all servers have completed running."""
    try:
        if self.children:
            self.wait_on_children()
        else:
            self.pool.waitall()
    except KeyboardInterrupt:
        pass

```

天色晚了，明天继续补充和细化 :)
