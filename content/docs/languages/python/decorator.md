---
weight: 2
bookToc: true
title: "装饰器"
---

## 装饰器

装饰器本质是**一个函数**，调用该函数会返回**另一个函数**替换函数，替换函数会调用原函数，来实现对原函数横切功能的新增。

{{< hint info >}}
因为本质为函数替换，原函数的元信息会被丢失，不是我们想要的。通过引入`functools.wraps`解决。如果装饰器需要能够接收参数，我们需要**额外的一层函数调用**用于接收参数，并返回上述`my_decorator`。

> https://stackoverflow.com/questions/6392739/what-does-the-at-symbol-do-in-python
  https://stackoverflow.com/questions/308999/what-does-functools-wraps-do
{{< /hint >}}

{{< tabs "装饰器实现" >}}
{{< tab "原生" >}}
```python
def my_decorator(f): # 此函数是为了调用触发替换过程，返回替换函数，替换函数会调用原函数，并替换原函数
    # wrapper函数名随意，一般习惯用wrapper
    def wrapper(*args, **kwargs):
        print('Something is happening before the function is called.')
        return f(*args, **kwargs)
    return wrapper # 返回替换函数

@my_decorator
def sum(a,b):
    """sum a and b"""
    return a+b

# 以上syntax等价于
# sum = my_decorator(sum)

print(sum(1,2)) # Something is happening before the function is called. 3
print(sum.__name__) # wrapper
print(sum.__doc__) # None
```
{{< /tab >}}
{{< tab "functools.wraps" >}}
```python
from functools import wraps

def my_decorator(f): # 此函数是为了调用触发替换过程，返回替换函数，替换函数会调用原函数，并替换原函数
    @wraps(f) # 保留原函数的元信息
    # wrapper函数名随意，一般习惯用wrapper
    def wrapper(*args, **kwargs):
        print('Something is happening before the function is called.')
        return f(*args, **kwargs)
    return wrapper # 返回替换函数

def sum(a,b):
    """sum a and b"""
    return a+b

# 以上syntax等价于
# sum = my_decorator(sum)

print(sum(1,2)) # 3
print(sum.__name__) # sum
print(sum.__doc__) # sum a and b
```
{{< /tab >}}
{{< tab "functools.wraps with args" >}}
```python
from functools import wraps

def my_decorator_with_args(arg): # 此函数是为了接收装饰器参数
    def my_decorator(f): # 此函数是为了调用触发替换过程，返回替换函数，替换函数会调用原函数，并替换原函数
        @wraps(f) # 保留原函数的元信息
        # wrapper函数名随意，一般习惯用wrapper
        def wrapper(*args, **kwargs):
            print('Something is happening before the function is called.')
            print(f'decorator arg is {arg}')
            return f(*args, **kwargs)
        return wrapper # 返回替换函数
    return my_decorator

@my_decorator_with_args('zqq')
def sum(a,b):
    """sum a and b"""
    return a+b

# 以上syntax等价于
# sum = my_decorator_with_args('zqq')(sum)

print(sum(1,2)) # 3
print(sum.__name__) # sum
print(sum.__doc__) # sum a and b
```
{{< /tab >}}
{{< /tabs >}}