layout: post
title: proto 模式
categories: [DesignPattern]
description: Use a factory and clones of a prototype for new instances (if instantiation is expensive).
keywords: Python, DesignPattern, pool
---

proto 模式
代码参考引用自：https://github.com/faif/python-patterns/blob/master/prototype.py

#### 代码

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import copy

class Prototype:

    value = 'default'

    def clone(self, **attrs):
        """Clone a prototype and update inner attributes dictionary"""
        obj = copy.deepcopy(self)
        obj.__dict__.update(attrs)
        return obj


class PrototypeDispatcher:

    def __init__(self):
        self._objects = {}

    def get_objects(self):
        """Get all objects"""
        return self._objects

    def register_object(self, name, obj):
        """Register an object"""
        self._objects[name] = obj

    def unregister_object(self, name):
        """Unregister an object"""
        del self._objects[name]


def main():
    dispatcher = PrototypeDispatcher()
    prototype = Prototype()

    d = prototype.clone()
    a = prototype.clone(value='a-value', category='a')
    b = prototype.clone(value='b-value', category='b')
    dispatcher.register_object('objecta', a)
    dispatcher.register_object('objectb', b)
    print([{n: p.value} for n, p in dispatcher.get_objects.items()])


if __name__ == '__main__':
    main()

### OUTPUT ###
# [{'objectb': 'b-value'}, {'default': 'default'}, {'objecta': 'a-value'}]
```

#### 分析

该模式的目的为，在初始化某些对象的成本比较高时，改为通过克隆原型对象的方式创建新的对象。

具体实现方法为，在原型类中实现 `clone()` 方法，使用 `copy.deepcopy()` 方法克隆对象，并且复制原型对象的 `__dict__` 到新建对象的 `__dict__` 。使用 copy.deepcopy() 方法，对象中的子对象也会克隆。

PrototypeDispatcher 为可以用来克隆的原型对象的管理器，可以注册原型对象，删除注册原型对象。