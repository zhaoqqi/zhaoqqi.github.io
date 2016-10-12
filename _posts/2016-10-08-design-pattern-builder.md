---
layout: post
title: builder 模式
categories: [DesignPattern]
description: Instead of using multiple constructors, builder object receives parameters and returns constructed objects.
keywords: Python, DesignPattern, Builder
---

builder 模式
代码参考引用自：https://github.com/faif/python-patterns/blob/master/builder.py

#### 代码

```python
#!/usr/bin/python
# -*- coding : utf-8 -*-

"""
@author: Diogenes Augusto Fernandes Herminio <diofeher@gmail.com>
https://gist.github.com/420905#file_builder_python.py
"""

# Director
class Director(object):

    def __init__(self):
        self.builder = None

    def construct_building(self):
        self.builder.new_building()
        self.builder.build_floor()
        self.builder.build_size()

    def get_building(self):
        return self.builder.building

# Abstract Builder
class Builder(object):

    def __init__(self):
        self.building = None

    def new_building(self):
        self.building = Building()

    def build_floor(self):
        raise notImplementError

    def build_size(self):
        raise notImplementError

# Concrete Builder
class BuilderHouse(Builder):
    
    def build_floor(self):
        self.building.floor = 'One'

    def build_size(self):
        self.building.size = 'Big'

class BuilderFlat(Builder):
    
    def build_floor(self):
        self.building.floor = 'More than One'

    def build_size(self):
        self.building.size = 'Small'

# Product
class Building(object):

    def __init__(self):
        self.floor = None
        self.size = None

    def __repr__(self):
        return 'Floor: {0.floor} | Size: {0.size}'.format(self)

# Client
if __name__ == '__main__':
    director = Director()
    director.builder = BuilderHouse()
    director.construct_building()
    building = director.get_building()
    print(building)
    director.builder = BuilderFlat()
    director.construct_building()
    building = director.get_building()
    print(building)

### OUTPUT ###
# Floor: One | Size: Big
# Floor: More than One | Size: Small
```

#### 小结
Builder 模式的惯用介绍为：将复杂对象的创建同它们的具体表现形式（representation）区别开来，这样可以根据需要得到具有不同表现形式的对象。

从上面的代码来看就是，house 和 flat 两种 building 的创建细节被 Builder 类的具体实现类 BuilderHouse 和 BuilderFlat 的 `build_floor()` 和 `build_size()` 方法设置，由 Director 类的 `construct_building()` 方法隐藏创建细节。而最终产品 building 作为类 Director 的属性 Buidler 实例的属性，就可以根据 Director 类实例化时传入的 Builder 的具体实现类实例，用来创建和返回不同类型的 building 。从而实现了惯用介绍中说的 “将复杂对象的创建同它们的具体表现区别开来，可以根据需要得到具有不同表现形式的对象” 。
