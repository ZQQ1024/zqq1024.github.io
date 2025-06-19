---
weight: 2
bookToc: true
title: "元类"
---

## 元类

### 元类概念

像其他语言一样，class用于创建object instance，同时python引入了元类metaclass，用于创建class（class也是一种object）

{{< hint info >}}
所有默认类（没有显式地指定元类）的元类都是 type，type的元类是自身（type也是一种object，type(type)=type）。

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

### 元类例子

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
- **类实例化时**即class()时，会调用对应元类的`__call__`方法，类是元类的实例，类会作为元类`__call__`方法中的`self`参数，一般最终传递给`type.__call__`，`type.__call__`内部工作原理如下：
    - 调用`self.__new__`, 即`instance = A.__new__(A, *args, **kwargs)` 或 `object.__new__` 创建A的实例
    - 调用`self.__init__`，即`A.__init__(instance, *args, **kwargs)` 或 `object.__init__` 初始化A的实例

### 元类的一些实际用途

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