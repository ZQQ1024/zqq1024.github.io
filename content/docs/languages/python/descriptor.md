---
weight: 6
bookToc: true
title: "描述符"
---

描述符可以用来控制对属性的访问行为，实现计算属性、懒加载属性、属性访问控制等功能

当使用点操作符如 `obj.attr` 访问一个对象的属性时：
1. 对应字节码`LOAD_ATTR` -> `PyObject_GetAttr` -> `tp->tp_getattro` 对应 Python `__getattribute__`
```python
# cpython/Python/ceval.c
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

# cpython/Objects/typeobject.c
    TPSLOT("__getattribute__", tp_getattr, NULL, NULL, ""), # 历史遗留
    ...
    TPSLOT("__getattribute__", tp_getattro, slot_tp_getattr_hook, wrap_binaryfunc, # tp_getattro 槽对应__getattribute__ C dispatcher 为 slot_tp_getattr_hook

PyTypeObject PyBaseObject_Type = {
    ...
    PyObject_GenericGetAttr,                    /* tp_getattro */ # object的C结构tp_getattro槽位指向了PyObject_GenericGetAttr
    ...
```
{{< hint info >}}
这里slot的概念就是将python中`__xxx__`方法对应成C结构里的`tp_xxx`槽位
{{< /hint >}}

2. `slot_tp_getattr_hook` dispatch：
- 当`__getattribute__()`被覆盖且没有定义`__getattr__()` 则走 `slot_tp_getattro`，就是调用`__getattribute__()`
- 定义`__getattr__()`则走`slot_tp_getattr_hook` dispatcher，无条件调用`__getattribute__()`，无覆盖重写的情况对应`PyObject_GenericGetAttr`；如果`__getattribute__()` raise AttributeError异常且也定义了`__getattr__()`则额外调用`__getattr__()`，相当于`__getattr__` 兜底
```python
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
```python
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

    # 1. 按照MRO查找类型及其基类中的数据描述符，如果存在调用描述符的__get__方法
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

    # 2. 查找实例字典（__slots__无实例字典，但是对应类有字典），如果存在，直接返回该属性的值
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

    # 3. 如果实例字典中没有同名属性，是否存在非数据描述符（仅实现了 __get__ 方法的描述符，__slots__本质也是描述符））。如果存在，则调用描述符的 __get__ 方法。
    if (f != NULL) {
        res = f(descr, obj, (PyObject *)Py_TYPE(obj));
        if (res == NULL && suppress &&
                PyErr_ExceptionMatches(PyExc_AttributeError)) {
            PyErr_Clear();
        }
        goto done;
    }

    # 4. 如果上述步骤均未找到属性，则在类型的 dict 以及其基类的 dict 中按照 MRO（Method Resolution Order，方法解析顺序）查找属性（在步骤1 `PyType_Lookup` 做的这个事情）

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


TODO 实例字典、__slots__ 数据/非数据描述符

然后用法

@property

bound method

class method

static method
