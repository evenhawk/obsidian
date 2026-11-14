---
title: Python 类与对象
tags:
  - python
  - 面向对象
  - 类
status: draft
updated: 2026-10-06
---

# Python 类与对象

**类（class）** 是面向对象编程里对一类事物的**抽象模板**，用来描述该类对象共同拥有的**属性（数据）和行为（方法）**。
面向对象编程通常先抽象出类，再通过实例化创建具体对象
对象有其特征,比如人有名字,性别,年龄等
这样的特征可以定义为属性
而人能做的事情在类里边被定义为方法(method),比如学习和运动
方法本质上也是函数，但通常与类或实例绑定
```python
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def introduce(self):
        print(f"我是 {self.name}，今年 {self.age} 岁")


# 实例化
person = Person("Alice", 18)
person.introduce()
```
***
## 类的特性

### 封装
把数据（属性）和操作数据的方法打包在类内部，隐藏内部实现细节，只对外暴露安全的访问接口；
外部不能随意直接修改内部成员，提高安全性与代码内聚。
### 继承
一个类（子类 / 派生类）可以复用另一个类（父类 / 基类）的属性和方法，子类还可以扩展自己新功能。
分为单继承、多继承，解决代码重复。
### 多态
同一个方法，不同子类实现不同行为；父类引用指向子类对象，调用同一个接口会执行子类逻辑。核心：**一个接口，多种实现**。
在继承的子类里定义专门的方法
### 抽象
提取事物共同特征形成类，只关心共性，忽略无关细节；通过抽象类 / 抽象方法定义规范，只声明方法，不写具体实现，强制子类完成实现。

当子类和父类满足“是一个（is-a）”关系时，可以使用继承复用父类能力
```Python
# 继承的示例
class Restaurant:  
    def __init__(self,restaurant_name,cuisine_type, number_served):  
        self.restaurant_name = restaurant_name  
        self.cuisine_type = cuisine_type  
        self.number_served = number_served  
  
    def describe_restaurant(self):  
        print(f"the name of restaurant is {self.restaurant_name}")  
        print(f"it is a {self.cuisine_type} restaurant")  
    def open_restaurant(self):  
        print(f'{self.restaurant_name} is open')  
    def set_number_served(self,number_served):  
        self.number_served = number_served  
    def increment_one_served(self):  
        self.number_served += 1  
  
class IceCreamStand(Restaurant):  
    def __init__(self,restaurant_name,number_served,flavors):  
        super().__init__(restaurant_name, "ice cream", number_served)  
        self.flavors = flavors  
  
    def show_all_flavors(self):  
        print(self.flavors)  
  
abq = IceCreamStand("name",0,["coffee","soup","c"])  
abq.show_all_flavors()
```
调用 `super().__init__(父类参数...)` 可以执行父类的初始化逻辑，参数顺序必须与父类定义一致
如果子类并不满足“是一个（is-a）”关系，优先考虑组合；如果父类设计不合理，应先重构父类
***
组合表示 has a 
继承表示 is a
