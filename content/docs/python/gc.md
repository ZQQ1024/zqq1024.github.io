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


## 循环引用

## 分代回收

## 缓存机制

small_number

string intern

free_list
