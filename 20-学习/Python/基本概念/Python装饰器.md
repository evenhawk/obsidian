---
title: Python 装饰器
tags:
  - python
  - 函数
  - 装饰器
status: draft
updated: 2026-10-06
---

# Python 装饰器

## 装饰器是什么

装饰器本质上是一个**高阶函数**：它接收一个函数作为参数，并返回一个新的包装函数。

> 一句话理解：在不修改原函数代码的前提下，给函数增加额外功能。

## 为什么需要装饰器

例如有 10 个函数都需要统计运行时间，如果不使用装饰器，就要在每个函数内部重复编写计时代码。装饰器可以把这些重复逻辑统一封装，再通过 `@装饰器名` 应用。

## 基本用法

```python
from functools import wraps


def my_decorator(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print("函数开始执行")
        result = func(*args, **kwargs)
        print("函数执行完毕")
        return result

    return wrapper


@my_decorator
def say_hello():
    print("Hello!")


say_hello()
```

输出：

```text
函数开始执行
Hello!
函数执行完毕
```

`@my_decorator` 等价于：

```python
say_hello = my_decorator(say_hello)
```

使用 `@wraps(func)` 可以保留原函数的 `__name__`、文档字符串等元数据。

## 装饰带参数的函数

`*args` 和 `**kwargs` 可以接收任意位置参数和关键字参数，再原样传给被装饰函数。

```python
from functools import wraps


def trace(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print(f"开始调用 {func.__name__}")
        result = func(*args, **kwargs)
        print(f"{func.__name__} 执行完毕，结果: {result}")
        return result

    return wrapper


@trace
def add(a, b):
    return a + b


add(3, 5)
```

## 带参数的装饰器

带参数的装饰器需要在普通装饰器外再包一层函数。

```python
from functools import wraps


def repeat(times):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            result = None
            for _ in range(times):
                result = func(*args, **kwargs)
            return result

        return wrapper

    return decorator


@repeat(3)
def greet(name):
    print(f"Hello, {name}!")


greet("Alice")
```

## 多个装饰器

多个装饰器按照“从下到上包装、从上到下执行”的顺序工作：

```python
@decorator_a
@decorator_b
def func():
    pass
```

等价于：

```python
func = decorator_a(decorator_b(func))
```

## 计时示例

```python
import time
from functools import wraps


def timer(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        try:
            return func(*args, **kwargs)
        finally:
            elapsed = time.perf_counter() - start
            print(f"{func.__name__} 耗时: {elapsed:.4f} 秒")

    return wrapper


@timer
def slow_function():
    time.sleep(1)
    print("包装函数采集运行时间，然后统计输出")
    return "Done"


slow_function()
```

把计时写在 `finally` 中，可以保证原函数即使抛出异常，也能记录已经消耗的时间。
