---
layout: post
title: 抽象工厂模式
categories: [DesignPattern]
description: use a generic function with specific factories.
keywords: Python, DesignPattern, Builder
---

抽象工厂模式
代码参考引用自：https://github.com/faif/python-patterns/blob/master/abstract_factory.py

#### 代码
```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

# http://ginstrom.com/scribbles/2007/10/08/design-patterns-python-style/

"""Implementation of the abstract factory pattern"""

import random

class PetShop:

    """A pet shop"""
    def __init__(self, animal_factory=None):
        self.factory = animal_factory

    def show_pet(self):
        """Creates and shows a pet using the abstract factory"""
        pet = self.factory.get_pet()
        print('We have a lovely {}'.format(pet))
        print('It says {}'.format(pet.speak()))
        print('We also have {}'.format(self.pet_factory.get_food()))

class Dog:
    
    def speak(self):
        print 'woof'

    def __str__(self):
        return 'Dog'

class Cat:
    
    def speak(self):
        print 'meow'

    def __str__(self):
        return 'Cat'

class DogFactory:
    
    def get_pet(self):
        return Dog()

    def get_food(self):
        return 'dog food'

class CatFactory:
    
    def get_pet(self):
        return Cat()

    def get_food(self):
        return 'cat food'

def get_factory():
    """Let's by dynamic!"""
    return random.choice([DogFactory, CatFactory])

if __name__ == '__main__':
    for i in range(3):
        shop = PetShot(get_factory())
        shop.show_pet()
        print('=' * 20)

### OUTPUT ###
# We have a lovely Dog
# It says woof
# We also have dog food
# ====================
# We have a lovely Dog
# It says woof
# We also have dog food
# ====================
# We have a lovely Cat
# It says meow
# We also have cat food
# ====================
```

#### 小结
DogFactory 和 CatFactory 为具体宠物类型的工厂，PetShot 作为 Dog 和 Cat 的抽象工厂。抽象工厂根据创建 PetShot 时传入的具体宠物工厂类型，生成对应的宠物产品。