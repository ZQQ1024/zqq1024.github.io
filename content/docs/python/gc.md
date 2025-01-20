---
weight: 4
bookToc: true
title: "内存管理"
---

Python提供了自动化的内存管理，内存空间的分配与释放都是由Python解释器在运行时自动进行的，以减轻程序员的工作负担并在一定程度上解决内存泄露的问题。

这里我们以`CPython v3.8.0`为例介绍CPython的内存管理，Python只是语言标准，不同的实现对内存管理不同。

CPython的内存管理有三个关键点：
- 引用计数
- 循环引用
- 分代回收

## 引用计数

引用计数反映有多少个变量引用到该对象。可以见得，当对象的引用计数值为0时，它的内存应该被释放掉。

每一个对象反应在CPython中就是一个`PyObject`结构体，其中`ob_refcnt`记录了变量的引用计数，定义如下：
```c
typedef struct _object {
    _PyObject_HEAD_EXTRA
    Py_ssize_t ob_refcnt;
    struct _typeobject *ob_type;
} PyObject;
```

### 新增引用计数

如**创建一个列表**，会初始化引用计数为1：
```c
PyObject *
PyList_New(Py_ssize_t size)
    -> op = PyObject_GC_New(PyListObject, &PyList_Type);
        ->  op = PyObject_INIT(op, tp);
            -> _Py_NewReference(op);
                -> Py_REFCNT(op) = 1;
```

**将对象添加到列表中**，会导致被添加对象引用计数+1：
```c
int
PyList_Append(PyObject *op, PyObject *newitem)
    -> static int
       app1(PyListObject *self, PyObject *v)
       -> Py_INCREF(v);
           -> static inline void _Py_INCREF(PyObject *op)
              {
                  _Py_INC_REFTOTAL;
                  op->ob_refcnt++;
              }
```

### 降低引用计数

如 `del a`，删除变量 a 对象的引用，原对象的引用计数会减少 1，del语句直接作用在变量上，而不是数据对象上

{{< hint info >}}
> https://docs.python.org/3/library/dis.html

opcode对应如下
```
DELETE_FAST(var_num)
Deletes local co_varnames[var_num].

DELETE_NAME(namei)
Implements `del name`, where namei is the index into co_names attribute of the code object. 
Code objects represent byte-compiled executable Python code, or bytecode.
```
{{< /hint >}}

```python
def example():
    a = [1, 2, 3]  # 创建一个列表对象，引用计数为 1
    b = a           # b 引用 a 指向的同一个对象，引用计数增加到 2
    del a           # 删除变量 a 对象的引用。这意味着 a 不再指向原来的对象，原对象的引用计数会减少 1。 这里opcode为DELETE_FAST
```
对应的c代码是
```c
case TARGET(DELETE_FAST):
    -> SETLOCAL(oparg, NULL);
        -> Py_XDECREF // 检查对象是否为 NULL
            -> Py_DECREF
                -> _Py_DECREF
                    if (--op->ob_refcnt != 0) {
                    }
                    else {
                        _Py_Dealloc(op); // 调用op->tp_dealloc
                    }
```

本质都是通过`Py_DECREF`降低引用计数，当引用计数为0时，触发对应object的`tp_dealloc`，对于list类型逻辑主要为减少列表中每个item的引用计数：
```c
PyTypeObject PyList_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "list",
    sizeof(PyListObject),
    0,
    (destructor)list_dealloc,                   /* tp_dealloc */
    ...

static void
list_dealloc(PyListObject *op)
{
    Py_ssize_t i;
    PyObject_GC_UnTrack(op);
    Py_TRASHCAN_BEGIN(op, list_dealloc)
    if (op->ob_item != NULL) {
        /* Do it backwards, for Christian Tismer.
           There's a simple test case where somehow this reduces
           thrashing when a *very* large list is created and
           immediately deleted. */
        i = Py_SIZE(op);
        while (--i >= 0) {
            Py_XDECREF(op->ob_item[i]);
        }
        PyMem_FREE(op->ob_item);
    }
    if (numfree < PyList_MAXFREELIST && PyList_CheckExact(op))
        free_list[numfree++] = op;
    else
        Py_TYPE(op)->tp_free((PyObject *)op);
    Py_TRASHCAN_END
}
```

{{< hint info >}}
从`Py_DECREF`实现可以看到，当引用计数为0时就会触发回收，这种回收时实时的。回收内存的时间被分摊到了平时，不需要等待特定时间集中处理。
{{< /hint >}}

## 循环引用 & 分代回收 

### 思想

引用计数也有明显的缺点，无法解决循环引用问题。即A和B相互引用而再没有外部引用A与B中的任何一个，**它们的引用计数都为1，但显然应该被回收**。CPython使用标记-清除（Mark—Sweep）算法，第一阶段会把所有的活动对象打上标记，第二阶段是把那些没有标记的对象非活动对象进行回收。

此标记-清楚的处理阶段，程序的执行会被暂停一段时间，可能会影响对实时性要求较高的应用程序，所有为了近一步提升效率，又引入了分代回收的概念。核心思想是，大部分对象生命周期较短，往往在其刚创建不久后就被回收，小部分对象生命周期较长，经过回收依然存活，则其大概率还会再后续的回收中存活。

所以根据对象的存活时间将其划分为不同的代，在CPython中，通常分为三代，generation 0-2，generation 2 执行频率低


### 实现

## 缓存机制

small_number

string intern

free_list
