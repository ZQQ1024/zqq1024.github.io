---
weight: 4
bookToc: true
title: "迭代器与生成器"
---

## 迭代器

迭代器(iterator)允许一个一个的访问集合（列表、元组、字典等）中的元素，并记住当前的访问位置。

使用built-in的函数`iter()`/`next()`和迭代器进行交互。也可使用常规for循环进行遍历。

```python
list=[1,2,3,4]
it = iter(list) # 创建迭代器对象
print(next(it)) # 输出迭代器的下一个对象 1
print(next(it)) # 2

for x in it:
    print(x) # 3 4
```

### 迭代器协议

一个Object可以通过实现迭代器协议变为迭代器，包含下面2种方法：
- `__iter__(self)`: 返回迭代器对象；循环开始的时候返回
- `__next__(self)`: 返回下一个元素；raise `StopIteration`表示没有元素


```python
from collections.abc import Iterable

class MyIterator():
  def __iter__(self):
    self.a = 1
    return self
  
  def __next__(self):
    if self.a <= 20:
      x = self.a
      self.a += 1
      return x
    else:
      raise StopIteration

it = iter(MyIterator())
print(isinstance(it, Iterable)) # True

for x in it:
  print(x) # 1 2 3 ... 18 19 20
```

## 生成器

使用`yield`关键字的函数称为生成器函数(generator function)，调用函数不会执行函数体而是返回生成器对象(generator object)，真正执行发生在 `next(gen) / gen.send(v)` 的时候。生成器对象实现了迭代器协议，能够像上面使用迭代器的方式使用。

生成器通过在`yield`语句处暂停/恢复函数的执行，来在迭代过程中逐步产生值，而不需要一次返回所有的结果。这种`lazy evaluation`的特性使得在处理大数据集的时候能够节省大量内存。

对比斐波那契数列实现，实现到`limit 10`。
{{< tabs "斐波那契数列" >}}
{{< tab "list版本" >}}
```python
limit = 10

fib = []
a, b = 0, 1

for _ in range(limit):
    fib.append(b)
    a, b = b, a+b

print(fib) # [1, 1, 2, 3, 5, 8, 13, 21, 34, 55]
```
{{< /tab >}}
{{< tab "生成器版本" >}}
```python
def fib(limit):
  a, b = 0, 1
  for i in range(limit):
    yield b
    a, b = b, a+b

limit = 10
fib_gi = fib(limit)
print(fib_gi) # <generator object fib at ...>

for i in fib_gi:
  print(i) # 1 1 2 3 5 8 13 21 34 55
```
{{< /tab >}}
{{< /tabs >}}

---

还是带着问题去看源码，以CPython`3.8.0`源码为例：

**1. 调用generator function为什么是返回generator，而不像正常函数执行函数体**



**2. generator依靠什么机制暂停运行/恢复运行的**

CPython会把每个函数的执行包装成一个`PyFrameObject`对象，用于保存一次调用所需的全部执行上下文，可以理解为是函数执行过程中的快照，一个函数的执行过程中不会新建多个 frame，而是一直在修改同一个 frame 的状态

```C
// cpython/Include/frameobject.h
typedef struct _frame {
    PyObject_VAR_HEAD
    struct _frame *f_back;      /* previous frame, or NULL */
    PyCodeObject *f_code;       /* code segment */
    PyObject *f_builtins;       /* builtin symbol table (PyDictObject) */
    PyObject *f_globals;        /* global symbol table (PyDictObject) */
    PyObject *f_locals;         /* local symbol table (any mapping) */
    PyObject **f_valuestack;    /* points after the last local */
    /* Next free slot in f_valuestack.  Frame creation sets to f_valuestack.
       Frame evaluation usually NULLs it, but a frame that yields sets it
       to the current stack top. */
    PyObject **f_stacktop;
    PyObject *f_trace;          /* Trace function */
    char f_trace_lines;         /* Emit per-line trace events? */
    char f_trace_opcodes;       /* Emit per-opcode trace events? */

    /* Borrowed reference to a generator, or NULL */
    PyObject *f_gen;

    int f_lasti;                /* Last instruction if called */
    /* Call PyFrame_GetLineNumber() instead of reading this field
       directly.  As of 2.3 f_lineno is only valid when tracing is
       active (i.e. when f_trace is set).  At other times we use
       PyCode_Addr2Line to calculate the line from the current
       bytecode index. */
    int f_lineno;               /* Current line number */
    int f_iblock;               /* index in f_blockstack */
    char f_executing;           /* whether the frame is still executing */
    PyTryBlock f_blockstack[CO_MAXBLOCKS]; /* for try and loop blocks */
    PyObject *f_localsplus[1];  /* locals+stack, dynamically sized */
} PyFrameObject;
```

frame对象中包含以下信息：
- `f_code`: 当前执行的`PyCodeObject`，对应一整个函数/模块/生成器/推导式编译后的字节码和常量表`co_consts`、名字表、行号表`co_lnotab`等
- `f_lasti`: 最后一次已经执行完成的那条字节码指令在该 `frame.f_code` 里的偏移位置，字节码层面的 program counter
- `f_lineno`: 把 `f_lasti` 通过 `frame.f_code` 的行号表映射回源码中的第几行
- `f_valuestack`: 值栈（操作数栈）的栈底指针，指向这块 value stack 的起始位置
- `f_stacktop;`: 指向值栈的栈顶位置(更准确：下一个可写槽位，栈顶的下一格)

{{< hint info >}}
CPython 的字节码解释器是栈机（stack machine），如下面代码：
```python
def g(a, b):
    yield a + b
```

对应字节码是：
- LOAD_FAST a : push(a)
- LOAD_FAST b : push(b)
- BINARY_OP + : pop(b), pop(a), push(a+b)
- YIELD_VALUE : pop栈顶为pop(a+b)
{{< /hint >}}

`ceval.c`中`_PyEval_EvalFrameDefault`解释器主循环有对`YIELD_VALUE`的`opcode`的处理：
```C
// cpython/Python/ceval.c
_PyEval_EvalFrameDefault
    ->
        case TARGET(YIELD_VALUE): {
            retval = POP();

            if (co->co_flags & CO_ASYNC_GENERATOR) {
                PyObject *w = _PyAsyncGenValueWrapperNew(retval);
                Py_DECREF(retval);
                if (w == NULL) {
                    retval = NULL;
                    goto error;
                }
                retval = w;
            }

            f->f_stacktop = stack_pointer;
            goto exit_yielding;
        }
    ...
    ->
exit_yielding:
    if (tstate->use_tracing) {
        if (tstate->c_tracefunc) {
            if (call_trace_protected(tstate->c_tracefunc, tstate->c_traceobj,
                                     tstate, f, PyTrace_RETURN, retval)) {
                Py_CLEAR(retval);
            }
        }
        if (tstate->c_profilefunc) {
            if (call_trace_protected(tstate->c_profilefunc, tstate->c_profileobj,
                                     tstate, f, PyTrace_RETURN, retval)) {
                Py_CLEAR(retval);
            }
        }
    }    

    /* pop frame */
exit_eval_frame:
    if (PyDTrace_FUNCTION_RETURN_ENABLED())
        dtrace_function_return(f);
    Py_LeaveRecursiveCall();
    f->f_executing = 0;
    tstate->frame = f->f_back;

    return _Py_CheckFunctionResult(NULL, retval, "PyEval_EvalFrameEx");
}
```
- `retval = POP();`弹出栈顶的值
- `exit_eval_frame`中的`tstate->frame = f->f_back;`将当前正在执行的frame改为调用者的frame
- `return _Py_CheckFunctionResult(NULL, retval, "PyEval_EvalFrameEx");`返回`retval`给调用者；`PyEval_EvalFrameEx`第三个参数只是字符串标签，不影响返回链路，仍然是 `_PyEval_EvalFrameDefault() `的返回值。

**以上对应了`yield`生成器函数挂起以及返回调用者值的功能实现。**

---

`next(gen)`恢复执行的实现位置大致如下（暂时不考虑gen.send(x)，下面协程会说），`next(gen)`的机器码如下：
```
LOAD_GLOBAL next
LOAD_FAST gen
CALL
RETURN_VALUE
```
`LOAD_GLOBAL`会先查 f_globals（模块全局字典），找不到再查 f_builtins（builtins 字典），next 是 builtin name，定义在 `builtins` 模块里。
```C
// cpython/Python/bltinmodule.c
static PyMethodDef builtin_methods[] = {
    ...
    {"__build_class__", (PyCFunction)(void(*)(void))builtin___build_class__,
     METH_FASTCALL | METH_KEYWORDS, build_class_doc},
    {"__import__",      (PyCFunction)(void(*)(void))builtin___import__, METH_VARARGS | METH_KEYWORDS, import_doc},
    {"next",            (PyCFunction)(void(*)(void))builtin_next,       METH_FASTCALL, next_doc},
    {"print",           (PyCFunction)(void(*)(void))builtin_print,      METH_FASTCALL | METH_KEYWORDS, print_doc},
    {"vars",            builtin_vars,       METH_VARARGS, vars_doc},
    {NULL,              NULL},
};

...

static PyObject *
builtin_next(PyObject *self, PyObject *const *args, Py_ssize_t nargs)
  -> res = (*it->ob_type->tp_iternext)(it);
    -> // cpython/Objects/genobject.c
       static PyObject *
       gen_iternext(PyGenObject *gen)
         -> gen_send_ex(gen, NULL, 0, 0);     
         -> static PyObject *
            gen_send_ex(PyGenObject *gen, PyObject *arg, int exc, int closing) // arg 是 gen.send(arg)
            ...
            PyFrameObject *f = gen->gi_frame;
            ...
            /* Generators always return to their most recent caller, not
             * necessarily their creator. */
            Py_XINCREF(tstate->frame);
            assert(f->f_back == NULL);
            f->f_back = tstate->frame;

            gen->gi_running = 1;
            gen->gi_exc_state.previous_item = tstate->exc_info;
            tstate->exc_info = &gen->gi_exc_state;
            result = PyEval_EvalFrameEx(f, exc);
            tstate->exc_info = gen->gi_exc_state.previous_item;
            gen->gi_exc_state.previous_item = NULL;
            gen->gi_running = 0;
```

`PyGenObject`主要是把函数调用的执行现场frame保存到了`gi_frame`。

```
_PyEval_EvalCodeWithName

compute_code_flags

ste->ste_generator
```




## 协程

生成器和协程是有关系的放一块讲

https://www.jmyjmy.top/2024-05-29_from-generator-to-asyncio/