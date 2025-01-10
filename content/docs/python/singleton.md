---
weight: 3
bookToc: true
title: "单例模式"
---

## 单例模式

单例模式确保一个类只有一个实例，多次初始化返回同一个实例。可以使用类装饰器或元类实现

{{< tabs "单例模式实现" >}}
{{< tab "类装饰器实现" >}}
```python
from functools import wraps

def singleton(cls):
    """单例类装饰器"""
    instances = {}

    @wraps(cls)
    def wrapper(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    
    return wrapper

@singleton    
class Foo():
    pass

# @singleton 等价于 Foo = singleton(Foo)
print(type(Foo)) # <class 'function'>，为wrapper函数，而不是Foo类

# 由于类本身是可调用的对象，返回一个函数来替代类在调用时的行为是可行的，wrapper函数提供了与类构造行为相同的逻辑
# 更本质的是：Foo() 其实就等价于 wrapper.__call__()
f1 = Foo()
f2 = Foo()
print(f1 is f2)  # True
```
{{< /tab >}}
{{< tab "元类实现" >}}
```python
import threading

class SingletonMeta(type):
    _instances = {}
    _lock =  threading.Lock()

    def __call__(cls, *args, **kwargs):
        # cls 为 Foo 类
        # Foo 类 为 SingletonMeta 的实例，类的实例可以访问到类属性，所以可以访问到 _instances 和 _lock
        with cls._lock:
            if cls not in cls._instances:
                cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]
    
class Foo(metaclass=SingletonMeta):
    pass

f1 = Foo()
f2 = Foo()
print(f1 is f2)  # True
```
{{< /tab >}}
{{< /tabs >}}