---
title: Python lambda 匿名函数
tags:
  - python
  - 基础
  - 函数
  - lambda
status: draft
updated: 2026-10-06
---

# Python lambda 匿名函数

lambda 函数通常用于编写简单的、单行的函数，通常在需要函数作为参数传递的情况下使用，例如在 map()、filter()、reduce() 等函数中。
## lambda 函数特点
***
* lambda函数是匿名的,没有函数名称,但可以赋值给变量引用
* lambda函数通常只包含一行代码,用来编写简单的函数

## 语法格式
***
格式如下
```Python
lambda 参数名称: 表达式
```
还可以没有参数
```python
f = lambda:print("hello")
print(f())

# 输出
hello
None
```

## filter()、map()、reduce() 函数介绍
***
## reduce()累计
**作用**:对可迭代对象的元素进行**累积计算**，最终得到一个单一结果。
下面是一个使用 reduce() 和 lambda 表达式演示如何计算一个序列的累积乘积：
```Python
from functools import reduce
 
numbers = [1, 2, 3, 4, 5]
 
# 使用 reduce() 和 lambda 函数计算乘积
product = reduce(lambda x, y: x * y, numbers)
 
print(product)  # 输出：120
```
在 Python 3 中，`reduce()` 需要从 `functools` 导入。它会遍历 numbers 列表，并使用 lambda 函数将累积的结果不断更新，最终得到了 1 * 2 * 3 * 4 * 5 = 120 的结果。
把乘法换成加法,也可以实现累加的效果

## map()映射
**作用**：对可迭代对象的**每个元素**执行指定函数，返回一个新的可迭代对象。
```Python
# 将列表中的每个数字平方
nums = [1, 2, 3, 4, 5]
squares = map(lambda x: x ** 2, nums)
print(list(squares))  # [1, 4, 9, 16, 25]

# 等价于列表推导式
squares = [x ** 2 for x in nums]
```
`map()`函数会返回一个迭代器,需要注意用`list()`来进行转换

## filter()过滤
**作用**：按条件**筛选**可迭代对象中的元素，只保留满足条件的。
```Python
# 筛选出偶数
nums = [1, 2, 3, 4, 5, 6]
evens = filter(lambda x: x % 2 == 0, nums)
print(list(evens))  # [2, 4, 6]

# 等价于列表推导式
evens = [x for x in nums if x % 2 == 0]
```
函数应返回 `True` 或 `False`,返回 `True` 的元素会被保留
返回值和`map()`一样为一个迭代器
