---
weight: 4
bookToc: true
title: "MRO"
---

## 概念

多继承场景下，如何找到一个类C重写方法在父类中的调用顺序？

C的父类（包含自身）按照从近到远的顺序进行排列称为C的线性化(`linearization`)，`Method Resolution Order (MRO)`是构造线性化的方法。
在python环境下，可以认为C的MRO和C的线性化是一个含义。

多继承场景下，MRO同时满足如下2个特性**是比较困难的**：
- `local precedence ordering`: C(B1, ..., Bn)，则在MRO中Bi也会按照继承列表中的顺序进行排列
- `monotonic`: 如果在C的线性化中，C1在C2之前，则在任何C的子类的线性化中，C1也应在C2之前。如果派生新类这一无害操作改变了MRO的顺序，这肯定是不好的

Python历史上至少存在三种不同的 MRO 算法：
1. 2.2之前老式类(classic)：深度优先，从左往右
2. 2.2引入新式类(Python 2.2 new-style)：不详（只针对新式类）
3. 2.3及之后版本(C3)：C3线性算法（只针对新式类）

## classic

Python在2.2之前版本只有旧式类，没有新式类，旧式类采用 **深度优先、从左到右** 的查找策略。classic的MRO算法只适用于旧式类。

实现：CPython 2.1 版本的 `classobject.c` 文件中的`class_lookup`，在给定类及其继承链中查找指定名称的属性
```C
static PyObject *
class_lookup(PyClassObject *cp, PyObject *name, PyClassObject **pclass)
{
	int i, n;
	PyObject *value = PyDict_GetItem(cp->cl_dict, name);
	if (value != NULL) {
		*pclass = cp;
		return value;
	}
	n = PyTuple_Size(cp->cl_bases);
	for (i = 0; i < n; i++) {
		/* XXX What if one of the bases is not a class? */
		PyObject *v = class_lookup(
			(PyClassObject *)
			PyTuple_GetItem(cp->cl_bases, i), name, pclass);
		if (v != NULL)
			return v;
	}
	return NULL;
}
```

针对以下菱形继承(diamond diagram)：

```python
class A:
    def save(self):
        print("A")

class B(A):
    pass

class C(A):
    def save(self):
        print("C")

class D(B, C):
    pass

d = D()
d.save() # A
```

关系图如下：
```
              class A:
                ^ ^  def save(self): ...
               /   \
              /     \
             /       \
            /         \
        class B     class C:
            ^         ^  def save(self): ...
             \       /
              \     /
               \   /
                \ /
              class D
```

对应的MRO为`D, B, A, C, A`，因递归实现未记录已经访问过的类，导致A会出现2次但是无影响（在第二个A找到的任何内容在第一个A时已经找到了）。

肉眼可以看出此时忽略了`C.save`，继承C变得没有意义。详细发现，此时的MRO不满足上述的`monotonic`特性：

```
C的MRO为：C, A # C在A前
D的MRO为：D, B, A, C # 此时C在A后面了
```

从实际来说，classic的MRO算法并没有产生特别大的影响，菱形继承在旧式类的继承关系中是不常见的。

但是对应新式类呢？所有新式类继承自`object`，这导致菱形继承在新式类的继承关系中又是很常见的。

## 2.2 new-style

因为新式类的引入同时需要解决上述菱形问题，python2.2对原本的MRO算法进行了修改。

描述：很难描述，不重要

实现：CPython 2.2 版本的 `typeobject.c` 文件中的`mro_internal`，由`PyType_Ready()`进行调用，存储在`tp_mro`字段，在python中可以通过`__mro__`属性进行访问（python2.1中并无mro的概念）
```c
// {"__mro__", T_OBJECT, offsetof(PyTypeObject, tp_mro), READONLY},

static int
mro_internal(PyTypeObject *type)
{
	PyObject *mro, *result, *tuple;

    // 是否元类为type
	if (type->ob_type == &PyType_Type) {
		result = mro_implementation(type);
	}
	else {
		static PyObject *mro_str;
        // 允许自定义元类在Python级别定义自己独特的MRO计算逻辑
		mro = lookup_method((PyObject *)type, "mro", &mro_str);
		if (mro == NULL)
			return -1;
		result = PyObject_CallObject(mro, NULL);
		Py_DECREF(mro);
	}
	if (result == NULL)
		return -1;
	tuple = PySequence_Tuple(result);
	Py_DECREF(result);
	type->tp_mro = tuple;
	return 0;
}

...

static PyObject *
mro_implementation(PyTypeObject *type)
{
	int i, n, ok;
	PyObject *bases, *result;

	bases = type->tp_bases;
	n = PyTuple_GET_SIZE(bases);
	result = Py_BuildValue("[O]", (PyObject *)type);
	if (result == NULL)
		return NULL;
	for (i = 0; i < n; i++) {
		PyObject *base = PyTuple_GET_ITEM(bases, i);
		PyObject *parentMRO;
		if (PyType_Check(base))
			parentMRO = PySequence_List(
				((PyTypeObject*)base)->tp_mro);
		else
			parentMRO = classic_mro(base);
		if (parentMRO == NULL) {
			Py_DECREF(result);
			return NULL;
		}
        /* XXX later -- for now, we cheat: "don't do that" */
		if (serious_order_disagreements(result, parentMRO)) {
			Py_DECREF(result);
			return NULL;
		}
        // 核心逻辑
		ok = conservative_merge(result, parentMRO);
		Py_DECREF(parentMRO);
		if (ok < 0) {
			Py_DECREF(result);
			return NULL;
		}
	}
	return result;
}
```

还是针对上述菱形继承问题：

{{< tabs "菱形继承问题" >}}
{{< tab "2.2 新式类" >}}
```python
class A(object):
    def save(self):
        print("A")

class B(A):
    pass

class C(A):
    def save(self):
        print("C")

class D(B, C):
    pass

print(D.__mro__)
# (<class '__main__.D'>, <class '__main__.B'>, <class '__main__.C'>, <class '__main__.A'>, <type 'object'>)
```
D的MRO为：`D, B, C, A`
{{< /tab >}}
{{< tab "2.2 旧式类" >}}
```python
class A:
    def save(self):
        print("A")

class B(A):
    pass

class C(A):
    def save(self):
        print("C")

class D(B, C):
    pass

import inspect
# (<class __main__.D at 0x55e6042a6458>, <class __main__.B at 0x55e6042bcd68>, <class __main__.A at 0x55e604314678>, <class __main__.C at 0x55e6042a5a88>)
print inspect.getmro(D)
```
D的MRO为：`D, B, A, C`，cpython 2.2中`classic_mro`相较于2.1`class_lookup`会去重
{{< /tab >}}
{{< /tabs >}}

新算法解决了菱形继承问题，**看起来似乎满足了**`monotonic`，但后续被证实依然[不满足上述的`monotonic`特性](https://mail.python.org/pipermail/python-dev/2002-October/029035.html)和`local precedence ordering`特性，新算法只存在了python2.2中，python2.3及后续的python3版本中使用了下面的C3进行替代

{{< tabs "new-style问题" >}}
{{< tab "不满足 monotonic" >}}
`Samuele Pedroni`提供了以下例子证明新算法不满足上述的`monotonic`特性：
```python
class A(object): pass
class B(object): pass
class C(object): pass
class D(object): pass
class E(object): pass
class K1(A,B,C): pass
class K2(D,B,E): pass
class K3(D,A):   pass
class Z(K1,K2,K3): pass

# (<class '__main__.K3'>, <class '__main__.D'>, <class '__main__.A'>, <type 'object'>)
print K3.__mro__

# (<class '__main__.Z'>, <class '__main__.K1'>, <class '__main__.K3'>, <class '__main__.A'>, <class '__main__.K2'>, <class '__main__.D'>, <class '__main__.B'>, <class '__main__.C'>, <class '__main__.E'>, <type 'object'>)
print Z.__mro__
```

K3的MRO为：`K3, D, A, O`
Z 的MRO为：`Z, K1, K3, A, K2, D, B, C, E, O`

K3的MRO中，D在A之前，但是到了K3的子类Z的MRO中，D却在A之后，不满足`monotonic`特性。
{{< /tab >}}
{{< tab "不满足 local precedence ordering" >}}
考虑以下新式类例子（python2.2环境下）：
```python
>>> F=type('Food',(),{'remember2buy':'spam'})
>>> E=type('Eggs',(F,),{'remember2buy':'eggs'})
>>> G=type('GoodFood',(F,E),{}) # under Python 2.3 this is an error!
```
继承图如下：
```
             O
             |
(buy spam)   F
             | \
             | E   (buy eggs)
             | /
             G

      (buy eggs or spam ?)
```
```python
>>> G.remember2buy
'eggs'
>>> G.__mro__
(<class '__main__.GoodFood'>, <class '__main__.Eggs'>, <class '__main__.Food'>, <type 'object'>)
```
这违背了`local precedence ordering`，因为`L(G) = G E F O`，F在E之后，但是G的基类列表`F E`中，F在E之前。
虽然从继承关系上看E比F更具体，但打破局部优先顺序是非常容易到处出错和违背直觉的。

对于上述继承关系的旧式类，是满足局部优先顺序特性的。
{{< /tab >}}
{{< /tabs >}}



## C3

C3算法发表在论文[A monotonic superclass linearization for Dylan](https://doi.org/10.1145/236337.236343)中，非python独创，同时满足了上述`local precedence ordering`和`monotonic`2个特性，python2.3及后续python3版本针对新式类采用了C3算法

先介绍一些符号：
```
C1 C2 ... CN，表示 由类组成的列表 [C1, C2, ... , CN]

头部head = C1，表示列表中的第一个元素

尾部tail = C2 ... CN，表示列表中除head的剩余元素

C + (C1 C2 ... CN) = C C1 C2 ... CN，表示2个列表 [C] 和 [C1, C2, ... ,CN] 的和
```

C的线性化表示为：C  + merge(各父母的线性化，父母列表)。
```
L[C(B1 ... BN)] = C + merge(L[B1] ... L[BN], B1 ... BN)

特别地，
当C无父类时，L[C] = C
当C只有一个父类B时，L[C(B)] = C + merge(L[B],B) = C + L[B]
```

**merge**操作定义如下：
- 取第一个列表的第一个元素作头部，即 L[B1][0]
- 如果这个头部不在其他任何列表的尾部，则将其加入类 C 的线性化顺序中，并从合并列表中移除它
- 否则查看下一个列表，重复上述操作，直到所有类都被移除
- 无法找到合适的头部，合并是无法完成，Python 2.3 会拒绝创建类 C 并抛出异常

```python
class A(object): pass
class B(object): pass
class C(object): pass
class D(object): pass
class E(object): pass
class K1(A,B,C): pass
class K2(D,B,E): pass
class K3(D,A):   pass
class Z(K1,K2,K3): pass
```

针对上述例子，C3算法的结果如下：
```
快速可得
L(A) = A O
L(B) = B O
L(C) = B O
L(D) = D O
L(E) = E O
```

```
L(K1) = K1 + merge(L(A), L(B), L(C), ABC)
      = K1 + merge(AO, BO, CO, ABC)
      = K1 + A + merge(O, BO, CO, BC) # O不满足，跳到BO，取头B，发现B不在其他所有列表的尾部
      = K1 + A + B + merge(O, O, CO, C)
      = K1 + A + B + C + merge(O, O, O)
      = K1 A B C O

L(K2) = K2 + merge(L(D), L(B), L(E), DBE)
      = K2 + merge(DO, BO, EO, DBE)
      = K2 + D + merge(O, BO, EO, BE)
      = K2 + D + B + merge(O, O, EO, E)
      = K2 + D + B + E + merge(O, O, O)
      = K2 D B E O

L(K3) = K3 + merge(L(D), L(A), DA)
      = K3 + merge(DO, AO, DA)
      = K3 + D + merge(O, AO, A)
      = K3 + D + A + merge(O, O)
      = K3 D A O

L(Z) = Z + merge(L(K1), L(K2), L(K3), K1K2K3)
     = Z + merge(K1ABCO, K2DBEO, K3DAO, K1K2K3)
     = Z + K1 + merge(ABCO, K2DBEO, K3DAO, K2K3)
     = Z + K1 + K2 + merge(ABCO, DBEO, K3DAO, K3)
     = Z + K1 + K2 + K3 + merge(ABCO, DBEO, DAO)
     = Z + K1 + K2 + K3 + D + merge(ABCO, BEO, AO)
     = Z + K1 + K2 + K3 + D + A + merge(BCO, BEO, O)
     = Z + K1 + K2 + K3 + D + A + B + merge(CO, EO, O)
     = Z + K1 + K2 + K3 + D + A + B + C + merge(O, EO, O)
     = Z K1 K2 K3 D A B C E O
```

与python 2.7.18环境下的执行结果一致
```python
# python 2.7.18
K1.__mro__
(<class '__main__.K1'>, <class '__main__.A'>, <class '__main__.B'>, <class '__main__.C'>, <type 'object'>)

K2.__mro__
(<class '__main__.K2'>, <class '__main__.D'>, <class '__main__.B'>, <class '__main__.E'>, <type 'object'>)

K3.__mro__
(<class '__main__.K3'>, <class '__main__.D'>, <class '__main__.A'>, <type 'object'>)

Z.__mro__
(<class '__main__.Z'>, <class '__main__.K1'>, <class '__main__.K2'>, <class '__main__.K3'>, <class '__main__.D'>, <class '__main__.A'>, <class '__main__.B'>, <class '__main__.C'>, <class '__main__.E'>, <type 'object'>)
```

实现：CPython 2.3.1 版本的 `typeobject.c` 文件中的`mro_internal->mro_implementation->pmerge`，实现了C3算法，代码略

## 最后

当继承关系自身存在模糊、二义的情形下，任何算法都不能做到满足上述特性，**这并不是算法的问题**。考虑以下例子：

```python
>>> O = object
>>> class X(O): pass
>>> class Y(O): pass
>>> class A(X,Y): pass
>>> class B(Y,X): pass

>>> class C(A,B): pass # ?
```
因为在A中 X在Y之前，但在B中 X又在Y之后，自身就存在矛盾，所以无法创建类C满足`monotonic`特性。python2.3及以上版本会报错（python2.2不会，会生成一个特定顺序这里为`CABXYO`）
```python
>>> class C(A,B): pass
...
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: Error when calling the metaclass bases
    Cannot create a consistent method resolution
order (MRO) for bases Y, X
```

`Method Resolution Order (MRO)`的名称虽然有`method`一词，但它不仅仅用于查找方法的顺序。MRO 实际上决定了类任何属性的解析顺序。

## 参考

> https://www.python.org/download/releases/2.2.3/descrintro/#mro  
https://www.python.org/download/releases/2.3/mro/  
https://mail.python.org/pipermail/python-dev/2002-October/029035.html  
https://python-history.blogspot.com/2010/06/method-resolution-order.html  
https://stackoverflow.com/questions/21657822/python-and-order-of-methods-in-multiple-inheritance  

