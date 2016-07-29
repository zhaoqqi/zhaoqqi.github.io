---
layout: post
title: Borg — 多个实例间共享状态
categories: [DesignPattern]
description: 
keywords: shared state, Borg
---

Borg模式 —— 多个实例间共享状态
代码参考引用自：https://github.com/faif/python-patterns/blob/master/borg.py

### 上代码
```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

class Borg:
    """定义类成员变量，在类的各个实例间共享"""
    __shared_state = {}

    def __init__(self):
        """将类成员变量self.__dict__的引用指向__shared_state，
           使self.__dict__变为类实例间共享状态。
           实例的属性self.state赋值后会放入self.__dict__，
           所以self.state也会变为实例间共享。"""
        self.__dict__ = self.__shared_state
        self.state = 'Init'

    def __str__(self):
        return self.state


class YourBorg(Borg):
    pass

if __name__ == '__main__':
    rm1 = Borg()
    rm2 = Borg()

    rm1.state = 'Idle'
    rm2.state = 'Running'

    print('rm1: {0}'.format(rm1))
    print('rm2: {0}'.format(rm2))

    rm2.state = 'Zombie'

    print('rm1: {0}'.format(rm1))
    print('rm2: {0}'.format(rm2))

    print('rm1 id: {0}'.format(id(rm1)))
    print('rm2 id: {0}'.format(id(rm2)))

    rm3 = YourBorg()

    print('rm1: {0}'.format(rm1))
    print('rm2: {0}'.format(rm2))
    print('rm3: {0}'.format(rm3))
```

### 输出结果
```python
{'state': 'Init'}
{'state': 'Init'}
rm1: Running
rm2: Running
rm1: Zombie
rm2: Zombie
rm1 id: 140049238365896
rm2 id: 140049238365968
{'state': 'Init'}
rm1: Init
rm2: Init
rm3: Init
```

### 总结
1. 借助使用类的属性(成员变量)可以实现类实例之间的变量内容共享。
2. 本文参考的代码中只是借助self.__dict__=self.__shared_state这种方式描述实例间共享变量的方法。
   但并不建议在实际中使用。因为所有的实例属性(比如增加self.name='xx')均会变为实例间共享的状态。
3. __init__方法中的self.__dict__ = self.__shared_state，在类实例化过程中，因为实例中不存在__shared_state，
   所以会将类的属性self.__shared_state的引用赋值给实例的self.__dict__。


### 待续
Borg还可以写成使用装饰器的方式实现的，留待后续补充……

