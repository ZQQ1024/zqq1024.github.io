---
weight: 1
bookToc: true
title: "魔术方法"
---


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

**异步相关：**
- `__await__`: 实现了`__await__`方法为`awaitable`对象。`await obj` 等价于 `it = obj.__await__()` 先取到一个 `iterator`，然后由外部的事件循环驱动结束，结束标志为触发`raise StopIteration(value)`/`return value`，这个 `value` 就是 `await obj` 的结果
```python
# 自己实现__await__，除此之外 coroutine，Task，Future也是awaitable的
class MyAwaitable:
    def __await__(self):
        print("[__await__] enter")
        # __await__ 必须返回 iterator。最常见就是返回一个生成器
        yield "tick-1"
        print("[__await__] resume 1")
        yield "tick-2"
        print("[__await__] resume 2")
        return 123

# 下面就是模拟 await x
def drive_awaitable(obj):
    it = obj.__await__()
    while True:
        try:
            x = it.send(None)      # 外部驱动：事件循环本质上就是在合适时机 send(None)
            print("[driver] got yield:", x)
        except StopIteration as e:
            print("[driver] finished with StopIteration.value =", e.value)
            return e.value

result = drive_awaitable(MyAwaitable())
print("final result:", result)
```

- `__aiter__`/`__anext__`: 异步迭代器协议，`__aiter__`返回一个异步迭代器，`__anext__`返回一个awaitable对象。对应搭配`async for`使用
```python
import asyncio

class Counter:
    def __init__(self, n):
        self.i = 0
        self.n = n

    def __aiter__(self):
        return self  # 自己就是异步迭代器

    async def __anext__(self):
        if self.i >= self.n:
            raise StopAsyncIteration
        print("yield control")
        await asyncio.sleep(1)  # 模拟异步 IO/等待
        v = self.i
        self.i += 1
        return v

async def main():
    # async for 必须在协程中使用
    async for x in Counter(3):
        print(x)
    # 大致等于以下代码
    # iterable = Counter(3)
    # aiter = iterable.__aiter__()          # 得到“异步迭代器”
    # while True:
    #     try:
    #         item = await aiter.__anext__()  # 每次取一个：注意这里有 await
    #     except StopAsyncIteration:
    #         break
    #     print(item)

asyncio.run(main())
```

- `__aenter__`/`__aexit__`: 异步上下文管理协议，对应搭配`async with`使用，2者都是返回 awaitable 对象，进入 / 退出阶段允许 await，用于异步打开/释放资源


**其他：**
- `__wrapped__`: 主要用于访问装饰器包装的函数的原始版本
- `__call__(self, ...)`: 使对象能够像函数一样被调用。当执行 `obj()` 时，会调用 `obj.__call__()`
- `__reversed__(self)`: 返回对象的反向迭代器，支持 `reversed(obj)`
- `__index__(self)`: 返回对象的整数索引，`obj[index]`/`range`
- `__copy__(self)`: 对象的浅复制（浅拷贝）
- `__deepcopy__(self, memo)`: 对象的深复制（深拷贝），memo字典存放已经被copy的对象，避免循环创建`memo[id(self)] = new_copy`，有点像`visited`字典