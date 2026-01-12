---
weight: 6
bookToc: true
title: "描述符"
---

描述符可以用来控制对属性的访问行为，实现计算属性、懒加载属性、属性访问控制等功能

## 属性查找顺序

当使用点操作符如 `obj.attr` 访问一个对象的属性(attribute lookup)时：
1. 对应字节码`LOAD_ATTR` -> `PyObject_GetAttr` -> `tp->tp_getattro` 对应 Python `__getattribute__`
```c
// cpython/Python/ceval.c
PyObject *name = GETITEM(names, oparg);
            PyObject *owner = TOP();
            PyObject *res = PyObject_GetAttr(owner, name);

# cpython/Objects/object.c
PyObject *
PyObject_GetAttr(PyObject *v, PyObject *name)
{
    PyTypeObject *tp = Py_TYPE(v);
    ...
    if (tp->tp_getattro != NULL)
            return (*tp->tp_getattro)(v, name);
    ...

// cpython/Objects/typeobject.c
    TPSLOT("__getattribute__", tp_getattr, NULL, NULL, ""), // 历史遗留
    ...
    TPSLOT("__getattribute__", tp_getattro, slot_tp_getattr_hook, wrap_binaryfunc, // tp_getattro 槽对应__getattribute__ C dispatcher 为 slot_tp_getattr_hook

PyTypeObject PyBaseObject_Type = {
    ...
    PyObject_GenericGetAttr,                    /* tp_getattro */ // object的C结构tp_getattro槽位指向了PyObject_GenericGetAttr
    ...
```
{{< hint info >}}
这里slot的概念就是将python中`__xxx__`方法对应成C结构里的`tp_xxx`槽位
{{< /hint >}}

2. `slot_tp_getattr_hook` dispatch：
- 当`__getattribute__()`被覆盖且没有定义`__getattr__()` 则走 `slot_tp_getattro`，就是调用`__getattribute__()`
- 定义`__getattr__()`则走`slot_tp_getattr_hook` dispatcher，无条件调用`__getattribute__()`，无覆盖重写的情况对应`PyObject_GenericGetAttr`；如果`__getattribute__()` raise AttributeError异常且也定义了`__getattr__()`则额外调用`__getattr__()`，相当于`__getattr__` 兜底
```c
/* There are two slot dispatch functions for tp_getattro.

   - slot_tp_getattro() is used when __getattribute__ is overridden
     but no __getattr__ hook is present;

   - slot_tp_getattr_hook() is used when a __getattr__ hook is present.

   The code in update_one_slot() always installs slot_tp_getattr_hook(); this
   detects the absence of __getattr__ and then installs the simpler slot if
   necessary. */

static PyObject *
slot_tp_getattro(PyObject *self, PyObject *name)
{
    PyObject *stack[1] = {name};
    return call_method(self, &PyId___getattribute__, stack, 1);
}

...

static PyObject *
slot_tp_getattr_hook(PyObject *self, PyObject *name)
{
    PyTypeObject *tp = Py_TYPE(self);
    PyObject *getattr, *getattribute, *res;
    _Py_IDENTIFIER(__getattr__);

    /* speed hack: we could use lookup_maybe, but that would resolve the
       method fully for each attribute lookup for classes with
       __getattr__, even when the attribute is present. So we use
       _PyType_Lookup and create the method only when needed, with
       call_attribute. */
    getattr = _PyType_LookupId(tp, &PyId___getattr__);
    if (getattr == NULL) {
        /* No __getattr__ hook: use a simpler dispatcher */
        tp->tp_getattro = slot_tp_getattro;
        return slot_tp_getattro(self, name);
    }
    Py_INCREF(getattr);
    /* speed hack: we could use lookup_maybe, but that would resolve the
       method fully for each attribute lookup for classes with
       __getattr__, even when self has the default __getattribute__
       method. So we use _PyType_Lookup and create the method only when
       needed, with call_attribute. */
    getattribute = _PyType_LookupId(tp, &PyId___getattribute__);
    if (getattribute == NULL ||
        (Py_TYPE(getattribute) == &PyWrapperDescr_Type &&
         ((PyWrapperDescrObject *)getattribute)->d_wrapped ==
         (void *)PyObject_GenericGetAttr))
        res = PyObject_GenericGetAttr(self, name);
    else {
        Py_INCREF(getattribute);
        res = call_attribute(self, getattribute, name);
        Py_DECREF(getattribute);
    }
    if (res == NULL && PyErr_ExceptionMatches(PyExc_AttributeError)) {
        PyErr_Clear();
        res = call_attribute(self, getattr, name);
    }
    Py_DECREF(getattr);
    return res;
}
```
3. `PyObject_GenericGetAttr`转向了`_PyObject_GenericGetAttrWithDict`，其查找顺序如下：
```c
# cpython/Objects/object.c
PyObject *
PyObject_GenericGetAttr(PyObject *obj, PyObject *name)
{
    return _PyObject_GenericGetAttrWithDict(obj, name, NULL, 0);
}

* Generic GetAttr functions - put these in your tp_[gs]etattro slot. */

PyObject *
_PyObject_GenericGetAttrWithDict(PyObject *obj, PyObject *name,
                                 PyObject *dict, int suppress)
{
    /* Make sure the logic of _PyObject_GetMethod is in sync with
       this method.

       When suppress=1, this function suppress AttributeError.
    */

    PyTypeObject *tp = Py_TYPE(obj);
    PyObject *descr = NULL;
    PyObject *res = NULL;
    descrgetfunc f;
    Py_ssize_t dictoffset;
    PyObject **dictptr;

    if (!PyUnicode_Check(name)){
        PyErr_Format(PyExc_TypeError,
                     "attribute name must be string, not '%.200s'",
                     name->ob_type->tp_name);
        return NULL;
    }
    Py_INCREF(name);

    if (tp->tp_dict == NULL) {
        if (PyType_Ready(tp) < 0)
            goto done;
    }

    // 按照MRO查找类型及其基类中的**描述符**，如果存在且是**数据描述符**，则调用数据描述符的__get__方法
    descr = _PyType_Lookup(tp, name);

    f = NULL;
    if (descr != NULL) {
        Py_INCREF(descr);
        f = descr->ob_type->tp_descr_get;
        if (f != NULL && PyDescr_IsData(descr)) {
            res = f(descr, obj, (PyObject *)obj->ob_type);
            if (res == NULL && suppress &&
                    PyErr_ExceptionMatches(PyExc_AttributeError)) {
                PyErr_Clear();
            }
            goto done;
        }
    }

    // 查找实例字典，如果存在，直接返回该属性的值
    if (dict == NULL) {
        /* Inline _PyObject_GetDictPtr */
        dictoffset = tp->tp_dictoffset;
        if (dictoffset != 0) {
            if (dictoffset < 0) {
                Py_ssize_t tsize;
                size_t size;

                tsize = ((PyVarObject *)obj)->ob_size;
                if (tsize < 0)
                    tsize = -tsize;
                size = _PyObject_VAR_SIZE(tp, tsize);
                _PyObject_ASSERT(obj, size <= PY_SSIZE_T_MAX);

                dictoffset += (Py_ssize_t)size;
                _PyObject_ASSERT(obj, dictoffset > 0);
                _PyObject_ASSERT(obj, dictoffset % SIZEOF_VOID_P == 0);
            }
            dictptr = (PyObject **) ((char *)obj + dictoffset);
            dict = *dictptr;
        }
    }
    if (dict != NULL) {
        Py_INCREF(dict);
        res = PyDict_GetItemWithError(dict, name);
        if (res != NULL) {
            Py_INCREF(res);
            Py_DECREF(dict);
            goto done;
        }
        else {
            Py_DECREF(dict);
            if (PyErr_Occurred()) {
                if (suppress && PyErr_ExceptionMatches(PyExc_AttributeError)) {
                    PyErr_Clear();
                }
                else {
                    goto done;
                }
            }
        }
    }

    // 如果实例字典中没有同名属性，如果存在非数据描述符（仅实现了 __get__ 方法的描述符），则调用描述符的 __get__ 方法
    if (f != NULL) {
        res = f(descr, obj, (PyObject *)Py_TYPE(obj));
        if (res == NULL && suppress &&
                PyErr_ExceptionMatches(PyExc_AttributeError)) {
            PyErr_Clear();
        }
        goto done;
    }

    // 如果上述步骤均未命中，则在类型的 dict 以及其基类的 dict 中按照 MRO查找普通属性（在步骤1 `PyType_Lookup` 已经做了这个事情，只是还未使用，一次类查找，多次优先级判断）

    if (descr != NULL) {
        res = descr;
        descr = NULL;
        goto done;
    }

    if (!suppress) {
        PyErr_Format(PyExc_AttributeError,
                     "'%.50s' object has no attribute '%U'",
                     tp->tp_name, name);
    }
  done:
    Py_XDECREF(descr);
    Py_DECREF(name);
    return res;
}
```
- 按照MRO查找类型及其基类中的**描述符**，如果存在且是**数据描述符**，则调用数据描述符的__get__方法
- 查找实例字典，如果存在，直接返回该属性的值
- 如果实例字典中没有同名属性，如果存在非数据描述符（仅实现了 __get__ 方法的描述符），则调用描述符的 __get__ 方法
- 如果上述步骤均未命中，则在类型的 dict 以及其基类的 dict 中按照 MRO查找普通属性（在步骤1 `PyType_Lookup` 已经做了这个事情，只是还未使用，一次类查找，多次优先级判断）
- 如果都没找到抛`AttributeError`，则可能触发 `__getattr__`

## 实例字典 和 `__slots__`

只属于这个实例的属性空间。CPython 在对象内存布局里，给实例字典留了一个 `tp_dictoffset`，指向一块 `dict`
```python
class A:
    pass

a = A()
a.x = 10

print(a.__dict__) # {'x': 10}
```

使用 `__slots__` 通常没有实例字典，不能随意挂新属性，本质是在类里放了一组**数据描述符**，并在实例中分配固定偏移，但允许显示定义 `__dict__`
```python
class B:
    __slots__ = ("x",)

b = B()
b.x = 1

class C:
    __slots__ = ("x", "__dict__")

c = C()
c.y = 2     # OK
```


## 数据/非数据描述符

实现了描述符协议中的任意一个方法的对象就是一个描述符(descriptor)：
- `__get__(self, obj, type=None)`
- `__set__(self, obj, value)`
- `__delete__(self, obj)`

这些方法的参数含义如下：
- `self` 是当前定义的描述符对象实例
- `obj` 是该描述符将作用的对象实例
- `type` 是该描述符作用的对象的类型（即所属的类）

通过下面的例子理解参数的参与，`self`就是`Descriptor`的实例`x`，`obj`就是`Foo`的实例`foo`，`type`就是`Foo`类

通过类访问`Foo.x`仍然命中类字典里的描述符，但调用的是：`f.__get__(None, A)`，由描述符自己决定怎么处理 obj is None；**但描述符主要是为实例设计的**，因为只有实例访问时，才存在 `self`，才有实例字典 vs 描述符的**优先级竞争**，才能体现 data / non-data descriptor 的核心差异

```python
class Descriptor:

    def __get__(self, obj, type):
        # 描述符自己决定怎么处理 obj is None，即通过类的方式访问
        if obj is None:
            print('__get__(): Accessing x from the class', type)
            return self
        
        print('__get__(): Accessing x from the object', obj)
        return 'X from descriptor'

    def __set__(self, obj, value):
        print('__set__(): Setting x on the object', obj)
        obj.__dict__['_x'] = value

class Foo:
    x = Descriptor()

print(Foo.x)
# __get__(): Accessing x from the class <class '__main__.Foo'>
# <__main__.Descriptor object at 0x00000202A1A46F90>

foo = Foo()
print(foo.x)
# __get__(): Accessing x from the object <__main__.Foo object at 0x00000202A1A470E0>
# X from descriptor
```
---

根据所实现的协议方法不同，描述符又可分为两类：
- 若实现了 `__set__()` 或 `__delete__()` 任一方法，该描述符是一个数据描述符（data descriptor）。
- 若仅实现 `__get__()` 方法，该描述符是一个非数据描述符（non-data descriptor）。

两者的在表现行为上存在差异：
- 数据描述符总是会覆盖实例字典 `__dict__` 中的属性
- 而非数据描述可能会被实例字典 `__dict__` 中定义的属性所覆盖

---


## property

`property` 保证属性访问始终由代码逻辑控制，而不会被实例字典覆盖。如下例子，即使实例字典里有 'x'，也无法屏蔽 property。


{{< tabs "property" >}}
{{< tab "语法糖用法" >}}
```python
class A:
    def __init__(self):
        self._x = None

    @property
    def x(self):
        """I'm the 'x' property."""
        return self._x

    @x.setter
    def x(self, value):
        self._x = value

    @x.deleter
    def x(self):
        del self._x

a = A()

a.x = 1
a.__dict__['x'] = 999

print(a.x)
```
{{< /tab >}}
{{< tab "等价方式" >}}
property 本身是一个实现了描述符协议的类，它还可以通过以下等价方式使用
```python
class A:
    def __init__(self):
        self._x = None

    def getx(self):
        return self._x

    def setx(self, value):
        self._x = value

    def delx(self):
        del self._x

    x = property(getx, setx, delx, "I'm the 'x' property.")

a = A()

a.x = 1
a.__dict__['x'] = 999

print(a.x)
```
{{< /tab >}}
{{< /tabs >}}

需要明确的是`property`是**数据描述符**，这样才能保证属性查找时其优先级最高。

是否为数据描述符是根据`PyProperty_Type.tp_descr_set` 是否为 `NULL`判断的，而不是用户有没有写 `setter`，property 总是有 `tp_descr_get`/`tp_descr_set`，没写 `setter`只是 property 对象内部的 `fset` 是 `NULL`，但 `property_descr_set`这个入口仍然存在，走进去后再判断`fset`是否实现
```c
// cpython/Objects/descrobject.c
PyTypeObject PyProperty_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "property",                                 /* tp_name */
    ...
    0,                                          /* tp_dict */
    property_descr_get,                         /* tp_descr_get */
    property_descr_set,                         /* tp_descr_set */
    0,                                          /* tp_dictoffset */
    property_init,                              /* tp_init */
    PyType_GenericAlloc,                        /* tp_alloc */
    PyType_GenericNew,                          /* tp_new */
    PyObject_GC_Del,                            /* tp_free */
};
```

## bound method

我们定义的函数对象是一个**非数据描述符**实例，目的是让在类中所定义的函数在通过实例调用时成为绑定方法（bound method），能够在调用时自动携带正确的实例上下文

方法在调用时会自动传入对象实例作为第一个参数，这是方法和普通函数的区别
```python
class A:
  def f(self, x):
    return x

a = A()

print(A.f, a.f) #<function A.f at ...> <bound method A.f of <__main__.A object at ...>>
print(A.f(None, 2))
print(A.__dict__['f']) #<function A.f at ...>

# f 是一个描述符
# 通过A.f访问时，判断self 为None返回正常函数
# 通过a.f实例访问时，返回一个 MethodType 对象
# 变成了绑定方法调用，在调用时会自动将绑定的对象作为第一个参数
print(a.f(2))
bound_method = a.f
print(bound_method(2))
```
{{< hint info >}}
`<__main__.A object at ...>` 指的是 A 的某一个**实例**对象，且这个实例在当前进程中的内存地址标识
{{< /hint >}}

通过类访问`obj == NULL`返回正常函数`function`，通过实例访问返回`method`对象，CPython代码如下：
```c
// cpython/Objects/funcobject.c
/* Bind a function to an object */
static PyObject *
func_descr_get(PyObject *func, PyObject *obj, PyObject *type)
{
    if (obj == Py_None || obj == NULL) {
        Py_INCREF(func);
        return func;
    }
    return PyMethod_New(func, obj); // 本质是(function, instance) 的二元包装
}

// cpython/Include/classobject.h
typedef struct {
    PyObject_HEAD
    PyObject *im_func;   /* The callable object implementing the method */
    PyObject *im_self;   /* The instance it is bound to */
    PyObject *im_weakreflist; /* List of weak references */
    vectorcallfunc vectorcall;
} PyMethodObject;

// cpython/Objects/classobject.c

/* Method objects are used for bound instance methods returned by
   instancename.methodname. ClassName.methodname returns an ordinary
   function.
*/

PyObject *
PyMethod_New(PyObject *func, PyObject *self)
{
    PyMethodObject *im;
    if (self == NULL) {
        PyErr_BadInternalCall();
        return NULL;
    }
    im = free_list;
    if (im != NULL) {
        free_list = (PyMethodObject *)(im->im_self);
        (void)PyObject_INIT(im, &PyMethod_Type);
        numfree--;
    }
    else {
        im = PyObject_GC_New(PyMethodObject, &PyMethod_Type);
        if (im == NULL)
            return NULL;
    }
    im->im_weakreflist = NULL;
    Py_INCREF(func);
    im->im_func = func;
    Py_XINCREF(self);
    im->im_self = self;
    im->vectorcall = method_vectorcall;
    _PyObject_GC_TRACK(im);
    return (PyObject *)im;
}
```


## classmethod

`classmethod` 类似上面的`bound method`，走的是同一条路线，只是绑定的对象不同。

```python
class A:
    def f(self, x):
        return self, x

    @classmethod
    def g(cls, x):
        return cls, x

a = A()

print(A.__dict__['g']) # <classmethod(<function A.g at ...>)>

# a.g也是绑定A
print(A.g, a.g) # <bound method A.g of <class '__main__.A'>> <bound method A.g of <class '__main__.A'>>
```

`classmethod`忽略了`__get__(self, obj, type=None)`中的`obj`把`type`作为绑定对象，返回了一个绑定到类的`method`

```c
PyDoc_STRVAR(classmethod_doc,
"classmethod(function) -> method\n\
\n\
Convert a function to be a class method.\n\
\n\
A class method receives the class as implicit first argument,\n\
just like an instance method receives the instance.\n\
To declare a class method, use this idiom:\n\
\n\
  class C:\n\
      @classmethod\n\
      def f(cls, arg1, arg2, ...):\n\
          ...\n\
\n\
It can be called either on the class (e.g. C.f()) or on an instance\n\
(e.g. C().f()).  The instance is ignored except for its class.\n\
If a class method is called for a derived class, the derived class\n\
object is passed as the implied first argument.\n\
\n\
Class methods are different than C++ or Java static methods.\n\
If you want those, see the staticmethod builtin.");

PyTypeObject PyClassMethod_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "classmethod",
    ...
    Py_TPFLAGS_DEFAULT | Py_TPFLAGS_BASETYPE | Py_TPFLAGS_HAVE_GC,
    classmethod_doc,                            /* tp_doc */
    (traverseproc)cm_traverse,                  /* tp_traverse */
    (inquiry)cm_clear,                          /* tp_clear */
    0,                                          /* tp_richcompare */
    0,                                          /* tp_weaklistoffset */
    0,                                          /* tp_iter */
    0,                                          /* tp_iternext */
    0,                                          /* tp_methods */
    cm_memberlist,              /* tp_members */
    cm_getsetlist,                              /* tp_getset */
    0,                                          /* tp_base */
    0,                                          /* tp_dict */
    cm_descr_get,                               /* tp_descr_get */
    0,                                          /* tp_descr_set */
    offsetof(classmethod, cm_dict),             /* tp_dictoffset */
    cm_init,                                    /* tp_init */
    PyType_GenericAlloc,                        /* tp_alloc */
    PyType_GenericNew,                          /* tp_new */
    PyObject_GC_Del,                            /* tp_free */
};

static PyObject *
cm_descr_get(PyObject *self, PyObject *obj, PyObject *type)
{
    classmethod *cm = (classmethod *)self;

    if (cm->cm_callable == NULL) {
        PyErr_SetString(PyExc_RuntimeError,
                        "uninitialized classmethod object");
        return NULL;
    }
    if (type == NULL)
        type = (PyObject *)(Py_TYPE(obj));
    return PyMethod_New(cm->cm_callable, type);
}
```

## staticmethod

`staticmethod` 也是一个描述符，实现的效果是在属性访问时关闭上述绑定，不管我们通过实例调用还是通过类调用，最终都会调用原始的函数

```python
class E:
    @staticmethod
    def f(x):
        return x * 10

e = E()
print(E.f, e.f) # <function E.f at ...> <function E.f at ...>
print(E.__dict__['f']) # <staticmethod(<function E.f at ...>)>
print(E.__dict__['f'](3)) # 30
```

```c
/* A static method does not receive an implicit first argument.
   To declare a static method, use this idiom:

     class C:
         @staticmethod
         def f(arg1, arg2, ...):
             ...

   It can be called either on the class (e.g. C.f()) or on an instance
   (e.g. C().f()). Both the class and the instance are ignored, and
   neither is passed implicitly as the first argument to the method.

   Static methods in Python are similar to those found in Java or C++.
   For a more advanced concept, see class methods above.
*/

PyTypeObject PyStaticMethod_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "staticmethod",
    ...
    Py_TPFLAGS_DEFAULT | Py_TPFLAGS_BASETYPE | Py_TPFLAGS_HAVE_GC,
    staticmethod_doc,                           /* tp_doc */
    (traverseproc)sm_traverse,                  /* tp_traverse */
    (inquiry)sm_clear,                          /* tp_clear */
    0,                                          /* tp_richcompare */
    0,                                          /* tp_weaklistoffset */
    0,                                          /* tp_iter */
    0,                                          /* tp_iternext */
    0,                                          /* tp_methods */
    sm_memberlist,              /* tp_members */
    sm_getsetlist,                              /* tp_getset */
    0,                                          /* tp_base */
    0,                                          /* tp_dict */
    sm_descr_get,                               /* tp_descr_get */
    0,                                          /* tp_descr_set */
    offsetof(staticmethod, sm_dict),            /* tp_dictoffset */
    sm_init,                                    /* tp_init */
    PyType_GenericAlloc,                        /* tp_alloc */
    PyType_GenericNew,                          /* tp_new */
    PyObject_GC_Del,                            /* tp_free */
};

static PyObject *
sm_descr_get(PyObject *self, PyObject *obj, PyObject *type)
{
    staticmethod *sm = (staticmethod *)self;

    if (sm->sm_callable == NULL) {
        PyErr_SetString(PyExc_RuntimeError,
                        "uninitialized staticmethod object");
        return NULL;
    }
    Py_INCREF(sm->sm_callable);
    return sm->sm_callable;
}
```

`staticmethod`适用于只是想把函数放进类的命名空间，而不依赖、感知类。比如`工厂方法`需要通过类创建实例，则用`classmethod`；逻辑上属于这个类的纯函数，只是属于用`staticmethod`

**参考连接**

> [https://antocuni.eu/2025/08/25/inside-cpythons-attribute-lookup](https://antocuni.eu/2025/08/25/inside-cpythons-attribute-lookup)
[https://waynerv.com/posts/python-descriptor-in-detail/#%E6%8F%8F%E8%BF%B0%E7%AC%A6%E7%9A%84%E4%BD%9C%E7%94%A8](https://waynerv.com/posts/python-descriptor-in-detail/#%E6%8F%8F%E8%BF%B0%E7%AC%A6%E7%9A%84%E4%BD%9C%E7%94%A8)