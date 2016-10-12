layout: post
title: pool 模式
categories: [DesignPattern]
description: preinstantiate and maintain a group of instances of the same type.
keywords: Python, DesignPattern, pool
---

pool 模式
代码参考引用自：https://github.com/faif/python-patterns/blob/master/pool.py

#### 代码

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

"""http://stackoverflow.com/questions/1514120/python-implementation-of-the-object-pool-design-pattern"""

class QueueObject():

    def __init__(self, queue, auto_get=False):
        self._queue = queue
        self.object = self.queue.get() if auto_get else None

    def __enter__(self):
        if self.object is None:
            self.object = self.queue.get()
        return self.object

    def __exit__(self, Type, value, traceback):
        if self.object is not None:
            self._queue.put(self.object)
            self.object = None

    def __del__(self):
        if self.object is not None:
            self._queue.put(self.object)
            self.object = None

def main():
    try:
        import queue
    except ImportError:
        import Queue as queue

    def test_object(queue):
        queue_object = QueueObject(queue, True)
        print('Inside func: {}'.format(queue_object.object))

    sample_queue = queue.Queue()

    sample_queue.put('yam')
    with QueueObject(sample_queue) as obj:
        print('Inside func: {}'.format(obj))
    print('Outside func: {}'.format(sample_queue.get()))

    sample_queue.put('sam')
    test_object(sample_queue)
    print('Outside func {}'.format(sample_queue.get()))

    if not sample_queue.empty():
        print(sample_queue.get())

if __name__ == '__main__':
    main()

### OUTPUT ###
# Inside with: yam
# Outside with: yam
# Inside func: sam
# Outside func: sam
```

#### 分析

例子中的代码使用一个队列作为 Python 实例对象的“池”，用来提前创建，以及维护一些列待使用的对象。每次有使用这些对象需要的时候从池中获取目标对象，使用完成以后“池”会将对象回收以备下次使用。

`__enter__`方法和`__exit__`方法，可以在使用 QueueObject 类的时候使用 with 语句。`__del__`方法是在类的实例被删除时调用，该方法将 self.object 放入 self._queue 中，即放入了所传入参数的 queue 中。在“池”对象被删除的时刻，也完成了“池”对对象的回收动作。

该模式的目的是，提前在对象“池”中创建并维护一些列相同类型的 Python 对象。这些对象在后续的使用中不必实时创建，而是直接从“池”中获取，使用完之后再由“池”回收。

可以想到的应用场景：并发编程中的线程池，进程池；其他暂时想不到了。




