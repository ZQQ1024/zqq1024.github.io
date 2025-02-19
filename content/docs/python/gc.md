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

引用计数反映有多少个变量引用到该对象。当对象的引用计数值为0时，它的内存应该被释放掉。

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
从`Py_DECREF`实现可以看到，当引用计数为0时就会触发回收，这种回收时实时的。

回收内存的时间被分摊到了平时，不需要等待特定时间集中处理。
{{< /hint >}}

## 循环引用 & 分代回收 

### 思想

引用计数也有明显的缺点，无法解决循环引用问题。即A和B相互引用而再没有外部引用A与B中的任何一个，**它们的引用计数都为1，但显然应该被回收**。

```python
class A:
    def __init__(self):
        self.b = None

class B:
    def __init__(self):
        self.a = None

# 创建 A 和 B 的实例
a = A()
b = B()

# 让 A 和 B 互相引用
a.b = b
b.a = a
```


CPython gc使用标记-清除（Mark—Sweep）算法，第一阶段会把所有的活动对象(`reachable`)打上标记，第二阶段是把那些没有标记的对象非活动对象(`collectable`)进行回收。

此标记-清楚的处理阶段，程序的执行会被暂停一段时间`STW(Stop The World)`，可能会影响对实时性要求较高的应用程序。为了近一步提升效率，又引入了分代回收的概念。

分代回收的核心思想是，**大部分对象生命周期较短，往往在其刚创建不久后就被回收，少部分对象生命周期较长，如果经过回收后依然存活，则其大概率还会在再后续的回收中存活。**

所以根据对象的存活时间将其划分为不同的代，在CPython中，通常分为三代，generation 0-2。**新创建的对象加入generation 0，由generation 0-2扫描频率逐步降低，每次扫描后存活的对象落入下一代。**

### 实现

还是带着具体的问题去看源码，并尝试作出一些总结。gc的源码主要位于`cpython/Modules/gcmodule.c`

{{< tabs "问题&总结" >}}
{{< tab "1、新创建的对象如何纳入gc管理的？" >}}
新创建的container对象会通过`PyObject_GC_New`纳入gc的generation 0代，并触发generation 0代的count自增加1。

以创建列表`PyList_New`为例：

```c
op = PyObject_GC_New(PyListObject, &PyList_Type);
    -> op = PyObject_GC_New(PyListObject, &PyList_Type);
        -> _PyObject_GC_New -> _PyObject_GC_Malloc -> _PyObject_GC_Alloc
            -> g = (PyGC_Head *)PyObject_Malloc(size);
            -> state->generations[0].count++;
            -> if (state->generations[0].count > state->generations[0].threshold &&
            -> ...
                -> collect_generations(state);
            -> op = FROM_GC(g);
            -> return op;
    -> _PyObject_GC_TRACK(op); // 加入generation 0代双循环链表中
        -> _PyObject_GC_TRACK_impl
            -> PyGC_Head *last = (PyGC_Head*)(_PyRuntime.gc.generation0->_gc_prev);
               _PyGCHead_SET_NEXT(last, gc);
               _PyGCHead_SET_PREV(gc, last);
               _PyGCHead_SET_NEXT(gc, _PyRuntime.gc.generation0);
               _PyRuntime.gc.generation0->_gc_prev = (uintptr_t)gc;
    -> return (PyObject *) op;
```

CPython的gc模块通过`PyGC_Head`来跟踪container对象（先加入到 generation 0 代中），存储在`PyObject`之前，相当于给`PyObject`增加了一个头部信息（为了节省空间相关地址的低位bits会存储一些标记信息）：
```c
/* GC information is stored BEFORE the object structure. */
typedef struct {
    // Pointer to next object in the list.
    // 0 means the object is not tracked
    uintptr_t _gc_next;

    // Pointer to previous object in the list.
    // Lowest two bits are used for flags documented later.
    uintptr_t _gc_prev;
} PyGC_Head;

// Lowest bit of _gc_next is used for UNREACHABLE flag.
#define NEXT_MASK_UNREACHABLE  (1)

/* Bit flags for _gc_prev */
/* Bit 0 is set when tp_finalize is called */
#define _PyGC_PREV_MASK_FINALIZED  (1)
/* Bit 1 is set when the object is in generation which is GCed currently. */
#define _PyGC_PREV_MASK_COLLECTING (2)
/* The (N-2) most significant bits contain the real address. */ // 在 update_refs(young);  // gc_prev 也被临时 is used for gc_refs
#define _PyGC_PREV_SHIFT           (2)
#define _PyGC_PREV_MASK            (((uintptr_t) -1) << _PyGC_PREV_SHIFT)

/* Get an object's GC head */
#define AS_GC(o) ((PyGC_Head *)(o)-1)

/* Get the object given the GC head */
#define FROM_GC(g) ((PyObject *)(((PyGC_Head *)g)+1))
```

CPython中的我们说的generation 0-2，3代本质为`gc_generation`数组：
- 包含了一个head指针，指向双向循环链表头部，用于跟踪此代中的对象
- count 表示对象创建的数量或者年轻代的执行次数
- threshold 表示触发回收的阈值

存于`struct _gc_runtime_state *state`中：
```c
struct gc_generation {
    PyGC_Head head; // 双向循环链表
    int threshold; /* collection threshold */
    int count; /* count of allocations or collections of younger
                  generations */
};

struct gc_generation generations[NUM_GENERATIONS];
```

{{< /tab >}}
{{< tab "2、何时触发每一代的扫描？" >}}
```shell
>>> import gc
>>> gc.get_threshold()
(700, 10, 10)
```

每一代generation有count，threshold变量，默认generation 0-2 的阈值分别为700，10，10。

```c
void
_PyGC_Initialize(struct _gc_runtime_state *state)
{
    state->enabled = 1; /* automatic collection enabled? */

#define _GEN_HEAD(n) GEN_HEAD(state, n)
    struct gc_generation generations[NUM_GENERATIONS] = {
        /* PyGC_Head,                                    threshold,    count */
        {{(uintptr_t)_GEN_HEAD(0), (uintptr_t)_GEN_HEAD(0)},   700,        0},
        {{(uintptr_t)_GEN_HEAD(1), (uintptr_t)_GEN_HEAD(1)},   10,         0},
        {{(uintptr_t)_GEN_HEAD(2), (uintptr_t)_GEN_HEAD(2)},   10,         0},
    };
    for (int i = 0; i < NUM_GENERATIONS; i++) {
        state->generations[i] = generations[i];
    };
    state->generation0 = GEN_HEAD(state, 0);
    struct gc_generation permanent_generation = {
          {(uintptr_t)&state->permanent_generation.head,
           (uintptr_t)&state->permanent_generation.head}, 0, 0
    };
    state->permanent_generation = permanent_generation;
}
```

```c
state->generations[0].count++;
if (state->generations[0].count > state->generations[0].threshold &&
        state->enabled &&
        state->generations[0].threshold &&
        !state->collecting &&
        !PyErr_Occurred()) {
        state->collecting = 1;
        collect_generations(state);
        state->collecting = 0;
    }
```
新创建的container对象加入gc的generation 0代，generation 0 count ++，如果count > threshold，则触发`collect_generations(state);`垃圾回收，并进入下面的逻辑：
- 寻找oldest generation，需要count > threshold，记为i代，触发i代的回收
- i+1代 count += 1，小于等于i代的count = 0

```c
static Py_ssize_t
collect_generations(struct _gc_runtime_state *state)
{
    /* Find the oldest generation (highest numbered) where the count
     * exceeds the threshold.  Objects in the that generation and
     * generations younger than it will be collected. */
    Py_ssize_t n = 0;
    for (int i = NUM_GENERATIONS-1; i >= 0; i--) {
        if (state->generations[i].count > state->generations[i].threshold) {
            /* Avoid quadratic performance degradation in number
               of tracked objects. See comments at the beginning
               of this file, and issue #4074.
            */
            if (i == NUM_GENERATIONS - 1
                && state->long_lived_pending < state->long_lived_total / 4)
                continue;
            n = collect_with_callback(state, i);
                -> result = collect(state, generation, &collected, &uncollectable, 0);
                    ->
                        /* update collection and allocation counters */
                        if (generation+1 < NUM_GENERATIONS)
                            state->generations[generation+1].count += 1;
                        for (i = 0; i <= generation; i++)
                            state->generations[i].count = 0;

                        /* merge younger generations with one we are currently collecting */
                        for (i = 0; i < generation; i++) {
                            gc_list_merge(GEN_HEAD(state, i), GEN_HEAD(state, generation));
                        }
            break;
        }
    }
    return n;
}
```

总结就是：
- 通过每代的 count > threshold 触发扫描
- generation 0 的 count 表示的是新创建加入的对象数，其他 generation 表示的是上一代执行的次数
- 每一代被回收，会触发自身及之前所有代的 count 清零，以及`gc_list_merge`合并年轻的代，来实现对年轻代的回收

{{< /tab >}}
{{< tab "3、如何确定循环引用的？" >}}

循环引用只是不可达对象的一种特殊情况，gc并没有做特殊处理。

在 `gcmodule.c` 中，`update_refs` 和 `subtract_refs` 是两个关键函数
- `update_refs(young)`: 遍历所有容器对象，将对象的引用计数（`ob_refcnt`）复制到 `gc_prev` 中（此时为标记阶段， `gc_prev` 被用作 `gc_refs`，链表的完整性可以被暂时破坏，因为此时不需要遍历链表）
- `subtract_refs(young)`: 遍历所有容器对象，对于每个容器对象，遍历它内部引用的其他对象，对这些内部引用的对象，减少它们的 `gc_refs` 值。此操作结束后，`gc_refs > 0`，说明对象 reachable from outside，不能被回收。

通过 `update_refs` 和 `subtract_refs` 等通用机制来找到所有不可达的对象（unreachable），包括循环引用的对象。最终通过`delete_garbage(state, &unreachable, old);`进行回收。

{{< /tab >}}
{{< /tabs >}}

## 缓存机制

### small integers

CPython 对小整数（通常在 **[-5,257)** 左闭右开）进行了缓存。这些整数在程序运行期间会被预先分配并缓存，以避免频繁的内存分配和释放。当程序需要使用这些整数时，直接从缓存中获取，而不是创建新的对象。

```
>>> a = 1
>>> b = 1
>>> a is b
True
>>> a = 256
>>> b = 256
>>> a is b
True
>>> a = 257
>>> b = 257
>>> a is b
False
```

```c 
// cpython/Objects/longobject.c

#ifndef NSMALLPOSINTS
#define NSMALLPOSINTS           257
#endif
#ifndef NSMALLNEGINTS
#define NSMALLNEGINTS           5
#endif

#if NSMALLNEGINTS + NSMALLPOSINTS > 0
/* Small integers are preallocated in this array so that they
   can be shared.
   The integers that are preallocated are those in the range
   -NSMALLNEGINTS (inclusive) to NSMALLPOSINTS (not inclusive).
*/
static PyLongObject small_ints[NSMALLNEGINTS + NSMALLPOSINTS];
#ifdef COUNT_ALLOCS
Py_ssize_t _Py_quick_int_allocs, _Py_quick_neg_int_allocs;
#endif
```

### string intern

字符串驻留是针对字符串的缓存机制，既可发生在编译时期（如字符串字面量和变量名、函数名、类名等标识符的驻留），也可发生在运行时期（如通过 `sys.intern()` 手动驻留任意字符串）

针对编译时期，如果需要对字符串字面量进行驻留，字符串需满足`[a-zA-Z0-9_]*`正则
```
>>> a = "hello"
>>> b = "hello"
>>> a is b
True
>>> a = "hello@"
>>> b = "hello@"
>>> a is b
False
```

```c
/* all_name_chars(s): true iff s matches [a-zA-Z0-9_]* */
static int
all_name_chars(PyObject *o)
{
    const unsigned char *s, *e;

    if (!PyUnicode_IS_ASCII(o))
        return 0;

    s = PyUnicode_1BYTE_DATA(o);
    e = s + PyUnicode_GET_LENGTH(o);
    for (; s != e; s++) {
        if (!Py_ISALNUM(*s) && *s != '_')
            return 0;
    }
    return 1;
}
```

变量名、函数名、类名等标识符的驻留未看出有什么限制
```c
// Objects/codeobject.c
intern_strings(names);
intern_strings(varnames);
intern_strings(freevars);
intern_strings(cellvars);
intern_string_constants(consts); // 特殊的递归

/* Intern selected string constants */
static int
intern_string_constants(PyObject *tuple)
{
    int modified = 0;
    Py_ssize_t i;

    for (i = PyTuple_GET_SIZE(tuple); --i >= 0; ) {
        PyObject *v = PyTuple_GET_ITEM(tuple, i);
        if (PyUnicode_CheckExact(v)) {
            if (PyUnicode_READY(v) == -1) {
                PyErr_Clear();
                continue;
            }
            if (all_name_chars(v)) {
                PyObject *w = v;
                PyUnicode_InternInPlace(&v);
                if (w != v) {
                    PyTuple_SET_ITEM(tuple, i, v);
                    modified = 1;
                }
            }
        }
        else if (PyTuple_CheckExact(v)) {
            intern_string_constants(v);
        }
        else if (PyFrozenSet_CheckExact(v)) {
            PyObject *w = v;
            PyObject *tmp = PySequence_Tuple(v);
            if (tmp == NULL) {
                PyErr_Clear();
                continue;
            }
            if (intern_string_constants(tmp)) {
                v = PyFrozenSet_New(tmp);
                if (v == NULL) {
                    PyErr_Clear();
                }
                else {
                    PyTuple_SET_ITEM(tuple, i, v);
                    Py_DECREF(w);
                    modified = 1;
                }
            }
            Py_DECREF(tmp);
        }
    }
    return modified;
}
```

最终驻留的字符串存储会通过`PyUnicode_InternInPlace`驻留到全局的`interned`字典中
```c
// Objects/unicodeobject.c

void
PyUnicode_InternInPlace(PyObject **p)
{
    PyObject *s = *p;
    PyObject *t;
#ifdef Py_DEBUG
    assert(s != NULL);
    assert(_PyUnicode_CHECK(s));
#else
    if (s == NULL || !PyUnicode_Check(s))
        return;
#endif
    /* If it's a subclass, we don't really know what putting
       it in the interned dict might do. */
    if (!PyUnicode_CheckExact(s))
        return;
    if (PyUnicode_CHECK_INTERNED(s))
        return;
    if (interned == NULL) {
        interned = PyDict_New();
        if (interned == NULL) {
            PyErr_Clear(); /* Don't leave an exception */
            return;
        }
    }
    Py_ALLOW_RECURSION
    t = PyDict_SetDefault(interned, s, s);
    Py_END_ALLOW_RECURSION
    if (t == NULL) {
        PyErr_Clear();
        return;
    }
    if (t != s) {
        Py_INCREF(t);
        Py_SETREF(*p, t);
        return;
    }
    /* The two references in interned are not counted by refcnt.
       The deallocator will take care of this */
    Py_REFCNT(s) -= 2;
    _PyUnicode_STATE(s).interned = SSTATE_INTERNED_MORTAL;
}
```

### free list

空闲列表是一种对象复用机制，用于**减少内存分配和释放的开销**。

当对象被释放时，其内存不会被立即归还给操作系统，而是被添加到空闲列表中。当需要分配新对象时，首先从空闲列表中获取内存，而不是重新分配。

如创建和销毁列表：
```c
// Objects/listobject.c

/* Empty list reuse scheme to save calls to malloc and free */
#ifndef PyList_MAXFREELIST
#define PyList_MAXFREELIST 80
#endif

static PyListObject *free_list[PyList_MAXFREELIST]; // 存放释放对象空间列表
static int numfree = 0; // free_list当前个数

PyObject *
PyList_New(Py_ssize_t size)
{
    ...
    if (numfree) {
        numfree--;
        op = free_list[numfree];
        _Py_NewReference((PyObject *)op);
#ifdef SHOW_ALLOC_COUNT
        count_reuse++;
#endif
    } else {
        op = PyObject_GC_New(PyListObject, &PyList_Type);
        if (op == NULL)
            return NULL;
#ifdef SHOW_ALLOC_COUNT
        count_alloc++;
#endif
    }
    ...
}

static void
list_dealloc(PyListObject *op)
{
    ...
    if (numfree < PyList_MAXFREELIST && PyList_CheckExact(op))
        free_list[numfree++] = op;
    else
        Py_TYPE(op)->tp_free((PyObject *)op);
    ...
}
```

free list 会在`generation 2`触发回收时进行一次清除

```c
// Modules/gcmodule.c

/* Clear free list only during the collection of the highest
    * generation */
if (generation == NUM_GENERATIONS-1) {
    clear_freelists();
}

static void
clear_freelists(void)
{
    (void)PyMethod_ClearFreeList();
    (void)PyFrame_ClearFreeList();
    (void)PyCFunction_ClearFreeList();
    (void)PyTuple_ClearFreeList();
    (void)PyUnicode_ClearFreeList();
    (void)PyFloat_ClearFreeList();
    (void)PyList_ClearFreeList();
    (void)PyDict_ClearFreeList();
    (void)PySet_ClearFreeList();
    (void)PyAsyncGen_ClearFreeLists();
    (void)PyContext_ClearFreeList();
}
```