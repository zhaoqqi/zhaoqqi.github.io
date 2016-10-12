---
layout: post
title: 工厂方法模式
categories: [DesignPattern]
description: Factory method of design pattern.
keywords: Python, DesignPattern, factory method
---

工厂方法模式
代码参考引用自：https://github.com/zhaoqqi/python-patterns/blob/master/factory_method.py

#### 代码

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

class GreekGetter:
    
    """A simple localizer a la gettext"""
    def __init__(self):
        self.trans = dict(dog="σκύλος", cat="γάτα")

    def get(self, msgid):
        """We'll punt if we don't have a translation"""
        return self.trans.get(msgid, str(msgid))

class EnglishGetter:

    """Simply echoes the msg ids"""

    def get(self, msgid):
        return (str(msgid))

def get_localizer(language="English"):
    languages = dict(English=EnglishGetter, Greek=GreekGetter)
    return languages[language]()

# Create our localizers
e, g = get_localizer(language="English"), get_localizer(language="Greek")
# Localize some text
for msgid in "dog parrot cat bear".split():
    print (e.get(msgid), g.get(msgid))

### OUTPUT ###
# dog σκύλος
# parrot parrot
# cat γάτα
# bear bear
```

#### 小结
使用一个 function 实现简单工厂模式。根据出入 function 的参数，生成不同的类实例。
