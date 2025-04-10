---
weight: 1
bookToc: true
title: "语法基本"
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