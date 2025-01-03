---
weight: 1
bookToc: true
title: "python知识点整理"
---

## 关键字

关键字（keywords）是预定义的保留字，具有特殊意义。它们用于 Python 语言的语法结构中，并且不能用作变量名、函数名、类名等

{{< tabs "keywords" >}}
{{< tab "控制流" >}}
```python
a = 10

# 条件判断
if a > 10:
    print('a is greater than 10')
elif a < 10:
    print('a is less than 10')
else:
    print('a is equal to 10')

a = ["a","b","c",4,5]

# for循环
for i in a:
    if a == 4:
        continue
    print(i)

# while循环
while True:
    print('Infinite loop')
    break

# try-catch
try:
    print("Hello")
    raise Exception("An error")
except:
    print("An exception occurred")
finally:
    print("The 'try except' is finished")
```
{{< /tab >}}
{{< tab "命名" >}}
```python
# 函数
def test_fun():
    pass

# 类
class TestClass:
    def __init__(self, name):
        self.name = name
    
    def get_name(self):
        return self.name
    
t = TestClass('test')
print(t.get_name())

# 匿名函数 lambda 参数: 表达式
add = lambda x, y: x + y
print(add(3, 4))  # 输出 7
```
{{< /tab >}}
{{< tab "布尔与逻辑" >}}
```python
print(True and False) # False
print(True or False) # True
print(not True) # False
```
{{< /tab >}}
{{< tab "作用域" >}}
```python
a = 123

def modify_local():
    a = 456

modify_local()
print(a) # 123

def modify_global():
    # global 声明变量是来自全局作用域
    # 当你想在函数内部修改全局变量时，需要使用 global 声明该变量
    global a
    a = 456

modify_global()
print(a) # 456

def outer_function():
    b = 789
    def inner_function():
        # nonlocal 声明变量是非本地作用域，来自外部函数作用域
        # 外层函数没有该变量，nonlocal 会继续向上查找，直到找到最近的封闭作用域为止
        nonlocal b
        b = 101112
    inner_function()
    print(b)

outer_function() # 101112
```
{{< /tab >}}
{{< tab "异步I/O编程" >}}
```python
import asyncio

# 异步函数
async def async_task(name, delay):
    print(f"Task {name} started")

    # await 会暂停执行当前协程，不会阻塞当前线程，而是将控制权交还给事件循环，事件循环会继续执行其他任务。
    # 事件循环轮询检查任务状态，等到 await 后面的异步操作完成时会“唤醒”这个暂停的协程，继续执行协程中 await 后面的代码（该任务的回调）
    await asyncio.sleep(delay)  # 模拟异步 I/O 操作
    
    print(f"Task {name} finished after {delay} seconds")

# 主程序函数
async def main():
    # 启动多个异步任务
    await asyncio.gather(
        async_task("A", 2),
        async_task("B", 1),
        async_task("C", 3)
    )

# 启动事件循环，运行主程序
asyncio.run(main())
```
{{< /tab >}}
{{< tab "异常" >}}
```python
raise Exception("An error occurred") # This will raise an exception

x = 10
assert x > 10, "x should be greater than 10" # This will raise an AssertionError
```
{{< /tab >}}
{{< tab "导入" >}}
```python
import math as M
from math import pi as P

print(M.pi) # 3.141592653589793
print(P) # 3.141592653589793
```
{{< /tab >}}
{{< tab "生成器" >}}
```python
# 生成器是一个返回迭代器的函数，它通过 yield 表达式在执行过程中产生值
# 每次调用 yield，生成器函数都会暂停并返回一个值，同时保留其执行状态，直到下一次恢复执行
def count_up_to(max):
    count = 1
    while count <= max:
        yield count  # 每次生成一个数字并暂停
        count += 1

counter = count_up_to(5)

# 使用 next() 获取生成器中的值
print(next(counter))  # 输出 1
print(next(counter))  # 输出 2

# send的作用2个：
# 1. 发送一个值给生成器，控制yield表达式的返回值
# 2. 恢复生成器的执行，并继续到下一个 yield，相当于 next()
# 意思是如果yield左边有变量，那么send可以给这个变量赋值，这里左边没有变量，所以send的值没有意义
print(counter.send(10))  # 输出 3 
print(next(counter))  # 输出 4
print(next(counter))  # 输出 5
# print(next(counter))  # 会抛出 StopIteration 异常

def double_inputs():
    while True:
        x = yield
        yield x * 2

gen = double_inputs()
next(gen) # 预激生成器，暂停在 yield 语句上
print(gen.send(10))  # 输出 20
next(gen)
print(gen.send(6))  # 输出 12

gen = (x * 2 for x in range(5))  # 创建一个生成器表达式
for value in gen:
    print(value)  # 输出 0 2 4 6 8
```
{{< /tab >}}
{{< /tabs >}}

## built-in

`built-in`指的是一些不需要import或者定义就可以使用的函数、类型、模块、常数

**built-in**函数: `print()`/`len()`/`type()`/`int()`/`str()`/`float()`/`sum()`/`sorted()`/`max()`/`min()`/`abs()`/`range()`/`bytearray()`等

```python
print("Hello World")

print(int("123")) # 123
print(str(123)) # "123"
print(float("123.456")) # 123.456
print(abs(-123)) # 123

a = [1, 3, 2]
print(len(a)) # 3
print(type(a)) # <class 'list'>

print(sum(a)) # 6
print(sorted(a)) # [1, 2, 3]
print(max(a)) # 3
print(min(a)) # 1
print(range(5)) # range(0, 5)
print(list(range(5))) # [0, 1, 2, 3, 4]
```

{{< hint info >}}
sum、max、min都是针对iterator，range返回的是range 对象，实现了迭代器协议。

python3 中 print 为内建函数，python2 中为关键字
{{< /hint >}}

**built-in**类型: `int`/`float`/`str`/`bytes`/`bytearray`/`list`/`tuple`/`dict`/`set`/`bool`/`complex`/`None`等
```python
a = "Hello World"

print(type(a)) # <class 'str'>

print(type(a.encode('utf-8'))) # <class 'bytes'>

print(type(bytearray(a.encode('utf-8')))) # <class 'bytearray'>

print(bytearray(a.encode('utf-8'))[0]) # 72

print(ord('H')) # 72
```

{{< hint info >}}
list、dict、set、bytearray为mutable 类型（可原地更新，no hashable），其余为immutable 类型（不可原地更新， hashable可以作为dict的key）。
{{< /hint >}}

**built-in**常量：`None`/`True`/`False`等

**built-in**模块：
- `math`: `math.sqrt()` / `math.pi`
- `os`: `os.getcwd()` / `os.path`
- `sys`: `sys.argv`
- `datetime`: `datetime.now()`
- `random`: `random.randint()`
- `string`: `string.ascii_lowercase`

**built-in**异常：`IndexError`/`KeyError`等

## 魔术方法

魔术方法（Magic Methods）是具有特殊功能的一组方法，它们在特定的操作或事件发生时被自动调用。这些方法通常以双下划线（__）开头和结尾

**构造和初始化：**
- `__new__(cls)__`: 创建并返回新对象，`__new__` 先于 `__init__` 被调用，通常用于控制对象的创建过程
- `__init__(self, ...)`: 构造函数，用于控制对象的初始化过程。每当一个类被实例化时，`__init__` 会被自动调用
- `__del__(self)`: 析构函数，在对象被销毁之前调用。用于清理资源，如文件句柄、数据库连接等

{{< hint info >}}
`__new__` 是静态类方法，`__init__` 是实例方法，sefl作为参数，一般返回None。除了为了继承不可变类型如int、str、tuple或unicode类型外，一般不需要 override `__new__` 方法
{{< /hint >}}

**表示和打印：**
- `__str__(self)`: 返回对象的可打印字符串。被 `print()` 函数、`str()` 调用
- `__repr__(self)`: 返回对象的“官方”字符串*表示*。被 repr() 调用，**或在交互式解释器中显示对象时使用**
- `__len__(self)`: 返回对象的长度。当调用 len() 函数时，Python 会自动调用此方法
```bash
python
Python 2.7.18 (v2.7.18:8d21aa21f2, Apr 19 2020, 20:48:48)
[GCC 4.2.1 Compatible Apple LLVM 6.0 (clang-600.0.57)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> import uuid
>>> uuid.uuid1()
UUID('12729d40-c1d1-11ef-8c95-acde48001122')
>>> a = uuid.uuid1()
>>> print(a)
15403328-c1d1-11ef-8a1d-acde48001122
>>>
```

**比较操作：**
- `__eq__(self, other)`/`__ge__`/`__le__`/`__lt__`/`__gt__`/`__ne__`: `==`/`>=`/`<=`/`<`/`>`/`!=`

**算术操作：**
- `__add__(self, other)`/`__sub__`/`__mul__`: 实现加法 `+`/`-`/`*` 运算
- `__truediv__(self, other)`: 实现除法 `/` 运算
- `__floordiv__(self, other)`: 实现**地板除法** `//` 运算
- `__mod__(self, other)`: 实现取余 `%` 运算
- `__pow__(self, other)`: 实现幂运算 `**`
- `__matmul__(self, other)`: 实现矩阵乘法 `@`，第三方类似`numpy`支持
{{< hint info >}}
`//`为地板除法，返回不比原结果大的第一个整数，所以 `-4 // 3 == -2`，`4 // 3 == 1`
{{< /hint >}}

**容器协议：**
- `__getitem__(self, key)`: 获取索引 key 对应的元素，支持 `obj[key]` 语法
- `__setitem__(self, key, value)`: 设置索引 key 对应的元素，支持 `obj[key] = value` 语法
- `__delitem__(self, key)`: 删除索引 key 对应的元素，支持 `del obj[key]` 语法
- `__iter__(self)`: 返回一个迭代器对象
- `__next__(self)`: 返回迭代器的下一个元素
- `__contains__(self, item)`: 检查 item 是否在对象中，支持 `item in obj` 语法

**属性访问：**
- `__getattr__(self, name)`: 当访问对象中不存在的属性时被调用
- `__setattr__(self, name, value)`: 设置对象的属性时被调用
- `__delattr__(self, name)`: 删除对象的属性时被调用

```python
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __getattr__(self, name):
        print(f"Attribute '{name}' does not exist!")
        return f"'{name}' is not a valid attribute."

    def __setattr__(self, name, value):
        print(f"Setting attribute '{name}' to '{value}'")
        super().__setattr__(name, value)

person = Person("Alice", 30)

print(person.name)  # Alice
print(person.age)   # 30

print(person.gender)  # Attribute 'gender' does not exist! 
                      # 'gender' is not a valid attribute.

person.gender = 'male' # Setting attribute 'gender' to 'male'
```

**上下文管理器：**
- `__enter__(self)`: 在 with 语句块开始时调用
- `__exit__(self, exc_type, exc_val, exc_tb)`: 在 with 语句块结束时调用，用于处理异常或清理工作，`exc_type = exception class`/`exc_value = exception instance`/`exc_tb = error traceback`
{{< tabs "Context Manager 例子" >}}
{{< tab "文件" >}}
```python
with open("/etc/hosts", "r") as f:
    print(f.read())
```
{{< /tab >}}
{{< tab "线程锁" >}}
```python
import threading

# 全局计数器
counter = 0

# 创建一个锁对象
lock = threading.Lock()

# 定义一个线程要执行的任务
def increment_counter():
    global counter
    for _ in range(100000):
        with lock:  # 使用锁保护对共享资源的访问
            # 包含读取和修改2个非原子操作，不加锁会导致race conditions不同线程同时读取到相同的counter值，导致数据丢失
            counter += 1

# 创建多个线程
threads = []
for _ in range(5):  # 创建 5 个线程
    thread = threading.Thread(target=increment_counter)
    threads.append(thread)

# 启动所有线程
for thread in threads:
    thread.start()

# 等待所有线程完成
for thread in threads:
    thread.join()

print(f"Final counter value: {counter}") # 500000
```
{{< /tab >}}
{{< tab "数据库连接" >}}
```python
import sqlite3

with sqlite3.connect("test.db") as conn:
    # cursor 提供了一种遍历查询结果集的抽象机制，可以视为一个指针，每次指向多行数据的一行，就是类似终端中的光标
    # create table 一般不需要cursor，但是python需要，是一种概念上的overhead
    # https://stackoverflow.com/questions/6318126/why-do-you-need-to-create-a-cursor-when-querying-a-sqlite-database
    cursor = conn.cursor()
    cursor.execute("CREATE TABLE IF NOT EXISTS books (title TEXT, pages INTEGER)")
    cursor.execute("INSERT INTO books VALUES ('The Catcher in the Rye', 234)")
    cursor.execute("INSERT INTO books VALUES ('1984', 328)")
    cursor.execute("INSERT INTO books VALUES ('To Kill a Mockingbird', 281)")

    cursor.execute("SELECT * FROM books")
    books = cursor.fetchall()
    for book in books:
        print(book)
```
{{< /tab >}}
{{< tab "网络socket" >}}
```python
import socket

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.connect(("www.example.com", 80))
    s.sendall(b"GET / HTTP/1.1\r\nHost: www.example.com\r\n\r\n")
    response = s.recv(4096)
    print(response)
    print(response.decode("utf-8"))
```
{{< /tab >}}
{{< tab "目录遍历" >}}
```python
import os

with os.scandir(".") as entries:
    for entry in entries:
        print(entry.name)
```
{{< /tab >}}
{{< tab "自定义计时器" >}}
```python
import time

class Timer:
   def __enter__(self):
       self.start_time = time.time()
       return self

   def __exit__(self, exc_type, exc_value, traceback):
       self.end_time = time.time()
       elapsed_time = self.end_time - self.start_time
       print(f"Elapsed time: {elapsed_time} seconds")

with Timer() as timer:
    time.sleep(2)
# Elapsed time: 2.003740072250366 seconds
```
{{< /tab >}}
{{< /tabs >}}

**其他：**
- `__wrapped__`: 主要用于访问装饰器包装的函数的原始版本
- `__call__(self, ...)`: 使对象能够像函数一样被调用。当执行 `obj()` 时，会调用 `obj.__call__()`
- `__reversed__(self)`: 返回对象的反向迭代器，支持 `reversed(obj)`
- `__index__(self)`: 返回对象的整数索引，`obj[index]`/`range`
- `__copy__(self)`: 对象的浅复制（浅拷贝）
- `__deepcopy__(self, memo)`: 对象的深复制（深拷贝），memo字典存放已经被copy的对象，避免循环创建`memo[id(self)] = new_copy`，有点像`visited`字典


## 装饰器

装饰器本质是**一个函数**，调用该函数会返回**另一个函数**替换函数，替换函数会调用原函数；使用替换函数替换原函数，来实现对原函数横切功能的新增。

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

print(sum(1,2)) # 3
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

## 元类

像其他语言一样，class用于创建object instance，同时python引入了元类metaclass，用于创建class（class也是一种object）

{{< hint info >}}
所有默认类（没有显式地指定元类）的元类都是 type，type的元类是自身。

当显式地为类指定一个自定义元类时，**该类的元类就不是 type**。

但自定义元类通过继承自type实现，所以，**所有类都是type的instance**。（isinstance 函数用于检查一个对象是否是某个类或其子类的实例）
{{< /hint >}}


`type`的作用如下，用于获取instance对应的class。如果把类理解为instance，则对应的class即为该类的元类。
```python
>>> x = 5
>>> type(x)
<class 'int'>
>>> class Foo():
...     pass
...
>>> f = Foo()
>>> type(f)
<class '__main__.Foo'>
>>> type(Foo)
<class 'type'>
>>> type(type)
<class 'type'>
>>>
```

metaclass 和 inheritance 不同，以下代码B的父类是A，但A和B的元类都是`type`
```python
class A:
    pass

class B(A):
    pass

type(A) # <class 'type'>
type(B) # <class 'type'>
```

通过以下代码进一步理解`type`元类：
```python
class MyMeta(type):
    # cls 通常指代当前类本身，而 self 是实例的引用。
    
    # 这里的self为MyClass，args为"Alice"，元类的__call__一般self参数也写成cls
    def __call__(self, *args, **kwargs):
        print(f"Calling metaclass {self}")
        print(f"Args: {args}")

        # super() 用于访问当前类的父类（或者祖先类）的方法。不需要显式传递 self，self 会自动传递给目标方法
        # type.__call__ 方法用于创建类的实例，这里会调用 MyClass 的 __new__ / __init__ 方法
        instance = super().__call__(*args, **kwargs)
        # type.__call__ 会调用 self.__new__ 方法创建MyClass的实例，然后调用 __init__ 方法初始化实例
        return instance
    
    # 这里的cls为 MyMeta，name为 MyClass，bases为 MyClass 的父类，dct为 MyClass 的属性和方法
    def __new__(cls, name, bases, dct):
        print(f"Creating class {name}")
        return super().__new__(cls, name, bases, dct)
    
    def __init__(self, name, bases, dct):
        print(f"Initializing class {name}")
        super().__init__(name, bases, dct)

# 此时MyClass会被创建，MyClass为MyMeta的实例，MyMeta会调用__new__和__init__方法
# 实例() 会调用实例对应类的__call__方法，所以不会调用MyMeta（MyClass作为实例对应的类为MyMeta）的__call__方法
class MyClass(metaclass=MyMeta):
    def __init__(self, name):
        print(f"Initializing MyClass with name {name}")
        self.name = name
    
    def __call__(self, *args, **kwargs):
        print(f"Calling MyClass with name {self.name}")

# print(isinstance(MyClass, MyMeta))  # This is True, MyClass is an instance of MyMeta
# print(isinstance(MyClass, type))  # This is True, MyClass is an instance of type
# print(isinstance(MyMeta, type))  # This is True, MyMeta is an instance of type

# 此时会触发 MyMeta.__call__ 方法，传入的cls为MyClass，args为"Alice"
obj = MyClass("Alice")

print(type(obj))  # This is MyClass

# # 以下2行是等价的
# obj()  # This triggers MyClass.__call__
# type(obj).__call__(obj)
```
整理一下结论：
- `instance()` = `type(instance).__call__(instance)`，即`MyClass()` = `type(MyClass).__call__(MyClass)`
- `super()` 用于访问当前类的父类（或者祖先类）的方法。不需要显式传递 `self`，`self` 会自动传递给目标方法
- **类创建时**会调用对应元类的`__new__`和`__init__`方法
- **类实例化时**即class()时，会调用对应元类的`__call__`方法，类是元类的实例，类会作为元类`__call__`方法中的`self`参数，一般最终传递给`type.__call__`，**`type.__call__`内部工作原理如下：
    - 调用`self.__new__`, 即`instance = A.__new__(A, *args, **kwargs)` 或 `object.__new__` 创建A的实例
    - 调用`self.__init__`，即`A.__init__(instance, *args, **kwargs)` 或 `object.__init__` 初始化A的实例

**元类的一些实际用途**：
{{< tabs "元类用途" >}}
{{< tab "自动注册类" >}}
```python
# 定义元类
class RegistryMeta(type):
    registry = {}

    def __new__(cls, name, bases, attrs):
        new_class = super().__new__(cls, name, bases, attrs)
        cls.registry[name] = new_class
        return new_class

# 使用元类创建基类
class PluginBase(metaclass=RegistryMeta):
    pass

# 定义一些子类
class PluginA(PluginBase):
    pass

class PluginB(PluginBase):
    pass

# 查看注册表
print(RegistryMeta.registry)
```
```
{'PluginBase': <class '__main__.PluginBase'>, 'PluginA': <class '__main__.PluginA'>, 'PluginB': <class '__main__.PluginB'>}
```
{{< /tab >}}
{{< tab "强制类遵循接口" >}}
```python
# 定义元类
class InterfaceEnforcer(type):
    def __new__(cls, name, bases, attrs):
        required_methods = ['save', 'load']
        for method in required_methods:
            if method not in attrs:
                raise TypeError(f"Class '{name}' must implement '{method}' method.")
        return super().__new__(cls, name, bases, attrs)

# 使用元类创建基类
class DataModel(metaclass=InterfaceEnforcer):
    def save(self):
        pass

    def load(self):
        pass

# 正确实现的类
class User(DataModel):
    def save(self):
        print("User saved.")

    def load(self):
        print("User loaded.")

# 错误实现的类（缺少 'load' 方法）
try:
    class Product(DataModel):
        def save(self):
            print("Product saved.")
except TypeError as e:
    print(e)
```
```
Class 'Product' must implement 'load' method.
```
{{< /tab >}}
{{< tab "属性验证" >}}
```python
# 定义描述符用于验证
class PositiveInteger:
    def __init__(self, name):
        self.name = name
        self.private_name = f"_{name}"
    
    def __get__(self, instance, owner):
        return getattr(instance, self.private_name, None)
    
    def __set__(self, instance, value):
        if not isinstance(value, int) or value < 0:
            raise ValueError(f"{self.name} must be a positive integer.")
        setattr(instance, self.private_name, value)

# 定义元类
class ValidateMeta(type):
    def __new__(cls, name, bases, attrs):
        for key, value in attrs.items():
            if isinstance(value, int):
                attrs[key] = PositiveInteger(key)
        return super().__new__(cls, name, bases, attrs)

# 使用元类创建类
class Product(metaclass=ValidateMeta):
    price = 0
    quantity = 0

    def __init__(self, price, quantity):
        self.price = price
        self.quantity = quantity

# 创建实例
p = Product(100, 50)
print(p.price, p.quantity)

# 尝试设置无效值
try:
    p.price = -10
except ValueError as e:
    print(e)
```
```
100 50
price must be a positive integer.
```
{{< /tab >}}
{{< /tabs >}}

> https://stackoverflow.com/questions/100003/what-are-metaclasses-in-python  
https://realpython.com/python-metaclasses/  
https://www.reddit.com/r/learnprogramming/comments/119b6dd/how_the_call_works_in_meta_class/

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