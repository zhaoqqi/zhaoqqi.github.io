---
layout: post
title: Eventlet(2) - Hub
categories: [OpenStack, Python]
description: Eventlet hub.
keywords: OpenStack, Python, hub
---

#### Eventlet Hub

Eventlet 实现了多种类型的 Hub，在实际使用时，Eventlet 会根据系统运行环境选择最适合的一种实现方式。Eventlet 目前支持以下4种 Hub。

* epolls - 需要在 Linux 环境下，Python 2.6 或者 `python-poll` 包。它是处理速度最快的纯 Python Hub。
* poll - 需要在支持 poll 的平台上选择使用
* selects - 效率最低，适合任何平台。
* pyevent - 基于 libevent，效率最高。因为它不支持线程，所以默认不生效。

Hub 是 greenthread 执行IO任务的调度中心。greenthread 被创建出来后，被添加到 Hub 的 Timer 待调度列表中。在 Hub 的 MAINLOOP 检测到某 greenthread 有IO任务就绪时，切换到该 greenthread 执行其 function 方法。

```python
import eventlet

def test():
    print 'Hello Eventlet.'

eventlet.spawn(test)
eventlet.sleep(0)
```

运行结果：执行`sleep()`后，立即打印 `Hello Eventlet.`

`eventlet.spawn(test)` 创建了一个 greenthread，执行任务 test。

```python
# eventlet/greenthread.py

def spawn(func, *args, **kwargs):
    hub = hubs.get_hub()
    g = GreenThread(hub.greenlet)
    hub.schedule_call_global(0, g.switch, func, args, kwargs)
    return g
```

`hubs.get_hub()`获得线程内全局单例 hub 对象，使用`hub.greenlet`作为父协程创建 greenthread g， hub.greenlet=greenlet.greenlet(self.run)。`hub.schedule_call_global()`方法在 Hub 中注册`func`方法，`g.switch`切换到 greenthread g 的 `main()`方法执行。

```python
# eventlet/hubs/hub.py

class BaseHub(object):
    ......
    def __init__(self, clock=time.time):
        ......
        self.greenlet = greenlet.greenlet(self.run)
        ......

    def schedule_call_global(self, seconds, cb, *args, **kw):
        t = timer.Timer(seconds, cb, *args, **kw)
        self.add_timer(t)
        return t

    def add_timer(self, timer):
        scheduled_time = self.clock() + timer.seconds
        self.next_timers.append((scheduled_time, timer))
        return scheduled_time
```

真正触发切换到 greenthread g 执行的操作是 `eventlet.sleep(0)` 。

```python
# eventlet/greenthread.py

def sleep(seconds=0):
    """Yield control to another eligible coroutine until at least *seconds* have
    elapsed.

    *seconds* may be specified as an integer, or a float if fractional seconds
    are desired. Calling :func:`~greenthread.sleep` with *seconds* of 0 is the
    canonical way of expressing a cooperative yield. For example, if one is
    looping over a large list performing an expensive calculation without
    calling any socket methods, it's a good idea to call ``sleep(0)``
    occasionally; otherwise nothing else will run.
    """
    hub = hubs.get_hub()
    current = getcurrent()
    assert hub.greenlet is not current, 'do not call blocking functions from the mainloop'
    timer = hub.schedule_call_global(seconds, current.switch)
    try:
        hub.switch()
    finally:
        timer.cancel()
```

`hub.switch()` 执行 hub 的 `switch()` 方法。

```python
# eventlet/hubs/hub.py

def switch(self):
    cur = greenlet.getcurrent()
    assert cur is not self.greenlet, 'Cannot switch to MAINLOOP from MAINLOOP'
    switch_out = getattr(cur, 'switch_out', None)
    if switch_out is not None:
        try:
            switch_out()
        except:
            self.squelch_generic_exception(sys.exc_info())
    self.ensure_greenlet()
    try:
        if self.greenlet.parent is not cur:
            cur.parent = self.greenlet
    except ValueError:
        pass  # gets raised if there is a greenlet parent cycle
    clear_sys_exc_info()
    return self.greenlet.switch()
```

`self.greenlet.switch()` 切换到 self.greenlet， 即MAINLOOP 执行，MAINLOOP 即是 `hub.run()` 方法。

```python
# eventlet/hubs/hub.py

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

先捡重点的看，`prepare_timers()`和 `fire_timers()`。

```python
# # eventlet/hubs/hub.py

def prepare_timers(self):
    heappush = heapq.heappush
    t = self.timers
    for item in self.next_timers:
        if item[1].called:
            self.timers_canceled -= 1
        else:
            heappush(t, item)
    del self.next_timers[:]

def fire_timers(self, when):
    t = self.timers
    heappop = heapq.heappop

    while t:
        next = t[0]

        exp = next[0]
        timer = next[1]

        if when < exp:
            break

        heappop(t)

        try:
            if timer.called:
                self.timers_canceled -= 1
            else:
                timer()
        except self.SYSTEM_EXCEPTIONS:
            raise
        except:
            self.squelch_timer_exception(timer, sys.exc_info())
            clear_sys_exc_info()
```

`prepare_timers()` 将 `self.next_timers` 中没有 called 的 greenthread 放入 `self.timers`。`fire_timers()` 判断 `self.times` 中的 greenthread 到时间执行后，通过 `timer()` 的方式执行。

```python
# eventlet/hubs/timer.py

class Timer(object):
    ......
    def __call__(self, *args):
        if not self.called:
            self.called = True
            cb, args, kw = self.tpl
            try:
                cb(*args, **kw)
            finally:
                try:
                    del self.tpl
                except AttributeError:
                    pass
```

`cb` 就是 eventlet.spawn 中的 `g.switch`。那么执行 `cb(*args, **kw)` 就是切换到 g 中去执行。

```python


class GreenThread(greenlet.greenlet):
    
    def __init__(self, parent):
        greenlet.greenlet.__init__(self, self.main, parent)
        self._exit_event = event.Event()
        self._resolving_links = False

    def main(self, function, args, kwargs):
        try:
            result = function(*args, **kwargs)
        except:
            self._exit_event.send_exception(*sys.exc_info())
            self._resolve_links()
            raise
        else:
            self._exit_event.send(result)
            self._resolve_links()
```

切换到 g 后，执行 `main()` 函数，此时终于执行到了真正的 function 方法。
