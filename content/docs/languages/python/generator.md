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

编译阶段的`symtable`会对每一个block（模块/函数/类）构建一个 symtable_entry(ste)，符号表的作用是提供语义，如每个名字是`LOCAL/GLOBAL_IMPLICIT/GLOBAL_EXPLICIT`等，这个`namespace`是`generator`/`coroutine`/`async generator`等，提供给后续的字节码生成。

symtable 构建完成后，ste 不是平铺的，而是一棵树：
```
module (ste)
 ├─ function f (ste)
 │   ├─ lambda (ste)
 │   └─ list comprehension (ste)
 └─ class C (ste)
     └─ function method (ste)
```

符号表相关的结构体如下：
```C
struct _symtable_entry;

struct symtable {
    PyObject *st_filename;          /* name of file being compiled,
                                       decoded from the filesystem encoding */
    struct _symtable_entry *st_cur; /* current symbol table entry */
    struct _symtable_entry *st_top; /* symbol table entry for module */
    PyObject *st_blocks;            /* dict: map AST node addresses
                                     *       to symbol table entries */
    PyObject *st_stack;             /* list: stack of namespace info */
    PyObject *st_global;            /* borrowed ref to st_top->ste_symbols */
    int st_nblocks;                 /* number of blocks used. kept for
                                       consistency with the corresponding
                                       compiler structure */
    PyObject *st_private;           /* name of current class or NULL */
    PyFutureFeatures *st_future;    /* module's future features that affect
                                       the symbol table */
    int recursion_depth;            /* current recursion depth */
    int recursion_limit;            /* recursion limit */
};

typedef enum _block_type { FunctionBlock, ClassBlock, ModuleBlock }
    _Py_block_ty;

typedef struct _symtable_entry {
    PyObject_HEAD
    PyObject *ste_id;        /* int: key in ste_table->st_blocks */
    PyObject *ste_symbols;   /* dict: variable names to flags */
    PyObject *ste_name;      /* string: name of current block */
    PyObject *ste_varnames;  /* list of function parameters */
    PyObject *ste_children;  /* list of child blocks */
    PyObject *ste_directives;/* locations of global and nonlocal statements */
    _Py_block_ty ste_type;   /* module, class, or function */
    int ste_nested;      /* true if block is nested */
    unsigned ste_free : 1;        /* true if block has free variables */
    unsigned ste_child_free : 1;  /* true if a child block has free vars,
                                     including free refs to globals */
    unsigned ste_generator : 1;   /* true if namespace is a generator */
    unsigned ste_coroutine : 1;   /* true if namespace is a coroutine */
    unsigned ste_comprehension : 1; /* true if namespace is a list comprehension */
    unsigned ste_varargs : 1;     /* true if block has varargs */
    unsigned ste_varkeywords : 1; /* true if block has varkeywords */
    unsigned ste_returns_value : 1;  /* true if namespace uses return with
                                        an argument */
    unsigned ste_needs_class_closure : 1; /* for class scopes, true if a
                                             closure over __class__
                                             should be created */
    unsigned ste_comp_iter_target : 1; /* true if visiting comprehension target */
    int ste_comp_iter_expr; /* non-zero if visiting a comprehension range expression */
    int ste_lineno;          /* first line of block */
    int ste_col_offset;      /* offset of first line of block */
    int ste_opt_lineno;      /* lineno of last exec or import * */
    int ste_opt_col_offset;  /* offset of last exec or import * */
    struct symtable *ste_table;
} PySTEntryObject;
```

也就是在编译阶段的符号表阶段就确定了该block（函数）是否为生成器
```C
// cpython/Python/symtable.c
static int
symtable_visit_expr(struct symtable *st, expr_ty e)
    ...
    case Yield_kind:
        if (e->v.Yield.value)
            VISIT(st, expr, e->v.Yield.value);
        st->st_cur->ste_generator = 1;
        break;
    case YieldFrom_kind:
        VISIT(st, expr, e->v.YieldFrom.value);
        st->st_cur->ste_generator = 1;
        break;
```
**`generator function`的`ste_type = FunctionBlock`，`ste_coroutine = 0`，`ste_generator = 1`**

---

随后在编译阶段的字节码阶段`PyCodeObject.co_flags`被置为`CO_GENERATOR`
```C
static int
compute_code_flags(struct compiler *c)
{
    PySTEntryObject *ste = c->u->u_ste;
    int flags = 0;
    if (ste->ste_type == FunctionBlock) {
        flags |= CO_NEWLOCALS | CO_OPTIMIZED;
        if (ste->ste_nested)
            flags |= CO_NESTED;
        if (ste->ste_generator && !ste->ste_coroutine)
            flags |= CO_GENERATOR;
        if (!ste->ste_generator && ste->ste_coroutine)
            flags |= CO_COROUTINE;
        if (ste->ste_generator && ste->ste_coroutine)
            flags |= CO_ASYNC_GENERATOR;
        if (ste->ste_varargs)
            flags |= CO_VARARGS;
        if (ste->ste_varkeywords)
            flags |= CO_VARKEYWORDS;
    }

    /* (Only) inherit compilerflags in PyCF_MASK */
    flags |= (c->c_flags->cf_flags & PyCF_MASK);

    if ((c->c_flags->cf_flags & PyCF_ALLOW_TOP_LEVEL_AWAIT) &&
         ste->ste_coroutine &&
         !ste->ste_generator) {
        flags |= CO_COROUTINE;
    }

    return flags;
}
```

---

**函数调用最终返回generator对象，参看以下代码链路**
- `TARGET(CALL_FUNCTION)` -> `call_function` -> `_PyVectorcall_Function` -> `_PyFunction_Vectorcall` -> `_PyEval_EvalCodeWithName`

```C
// cpython/Python/ceval.c
PyObject* _Py_HOT_FUNCTION
_PyEval_EvalFrameDefault(PyFrameObject *f, int throwflag)
        case TARGET(CALL_FUNCTION): {
            PREDICTED(CALL_FUNCTION);
            PyObject **sp, *res;
            sp = stack_pointer;
            res = call_function(tstate, &sp, oparg, NULL);
            stack_pointer = sp;
            PUSH(res);
            if (res == NULL) {
                goto error;
            }
            DISPATCH();
        }

Py_LOCAL_INLINE(PyObject *) _Py_HOT_FUNCTION
call_function(PyThreadState *tstate, PyObject ***pp_stack, Py_ssize_t oparg, PyObject *kwnames)
{
    PyObject **pfunc = (*pp_stack) - oparg - 1;
    PyObject *func = *pfunc;
    PyObject *x, *w;
    Py_ssize_t nkwargs = (kwnames == NULL) ? 0 : PyTuple_GET_SIZE(kwnames);
    Py_ssize_t nargs = oparg - nkwargs;
    PyObject **stack = (*pp_stack) - nargs - nkwargs;

    if (tstate->use_tracing) {
        x = trace_call_function(tstate, func, stack, nargs, kwnames);
    }
    else {
        x = _PyObject_Vectorcall(func, stack, nargs | PY_VECTORCALL_ARGUMENTS_OFFSET, kwnames);
    }

    assert((x != NULL) ^ (_PyErr_Occurred(tstate) != NULL));

    /* Clear the stack of the function object. */
    while ((*pp_stack) > pfunc) {
        w = EXT_POP(*pp_stack);
        Py_DECREF(w);
    }

    return x;
}

// cpython/Include/cpython/abstract.h
static inline vectorcallfunc
_PyVectorcall_Function(PyObject *callable)
{
    PyTypeObject *tp = Py_TYPE(callable);
    Py_ssize_t offset = tp->tp_vectorcall_offset;
    vectorcallfunc *ptr;
    if (!PyType_HasFeature(tp, _Py_TPFLAGS_HAVE_VECTORCALL)) {
        return NULL;
    }
    assert(PyCallable_Check(callable));
    assert(offset > 0);
    ptr = (vectorcallfunc*)(((char *)callable) + offset);
    return *ptr;
}

// cpython/Objects/funcobject.c
...
op->vectorcall = _PyFunction_Vectorcall;
...

// cpython/Objects/call.c
PyObject *
_PyFunction_Vectorcall(PyObject *func, PyObject* const* stack,
                       size_t nargsf, PyObject *kwnames)
{
    ...
    return _PyEval_EvalCodeWithName((PyObject*)co, globals, (PyObject *)NULL,
                                    stack, nargs,
                                    nkwargs ? _PyTuple_ITEMS(kwnames) : NULL,
                                    stack + nargs,
                                    nkwargs, 1,
                                    d, (int)nd, kwdefs,
                                    closure, name, qualname);
}

// cpython/Python/ceval.c
PyObject *
_PyEval_EvalCodeWithName(PyObject *_co, PyObject *globals, PyObject *locals,
    ...
    /* Handle generator/coroutine/asynchronous generator */
    if (co->co_flags & (CO_GENERATOR | CO_COROUTINE | CO_ASYNC_GENERATOR)) {
        PyObject *gen;
        int is_coro = co->co_flags & CO_COROUTINE;

        /* Don't need to keep the reference to f_back, it will be set
         * when the generator is resumed. */
        Py_CLEAR(f->f_back);

        /* Create a new generator that owns the ready to run frame
         * and return that as the value. */
        if (is_coro) {
            gen = PyCoro_New(f, name, qualname);
        } else if (co->co_flags & CO_ASYNC_GENERATOR) {
            gen = PyAsyncGen_New(f, name, qualname);
        } else {
            gen = PyGen_NewWithQualName(f, name, qualname);
        }
        if (gen == NULL) {
            return NULL;
        }

        _PyObject_GC_TRACK(f);

        return gen;
    }
```


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
CALL_FUNCTION
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
            ...
            return result;
```

生成器函数第一次被调用时，并不像普通函数那样执行函数体，而是创建一个生成器对象，把将来要执行的frame准备好挂在 `gi_frame` 上，next时再重新执行frame。


## 协程

首先给协程coroutine下一个定义：协程是一个函数，可以暂停/恢复执行。协程中的co代表`cooperative`，指的是协程主动`yield control`而不是被外部`preempt`的。

因为协程的上述特性，很适合搭配外部的scheduler(event loop)进行自主控制的调度和non-blocking I/O，来处理高并发的I/O密集型任务，大致过程如下：
1. 把 fd 设成 O_NONBLOCK
2. 协程调用 recv/read
3. 如果立刻读到数据：直接返回，不挂起
4. 如果返回 -1 且 errno == EAGAIN/EWOULDBLOCK：说明“现在没数据”，在这里挂起协程，并把这个 fd 注册到event loop里监听“可读”事件
5. event loop等到 fd 变“可读”，把协程唤醒
6. 协程恢复后再 recv/read 一次
7. 循环直到读完/再次 EAGAIN

因为是在一个线程中运行多个协程，无法使用blocking的IO，不然会导致整个线程卡住。

可以看出协程的概念和生成器很像，Python对协程的支持本质也是基于生成器实现的。

> [https://www.jmyjmy.top/2024-05-29_from-generator-to-asyncio/](https://www.jmyjmy.top/2024-05-29_from-generator-to-asyncio/)  
[https://peps.python.org/pep-0342/](https://peps.python.org/pep-0342/)  
[https://peps.python.org/pep-0380/](https://peps.python.org/pep-0380/)  
[https://peps.python.org/pep-0492/](https://peps.python.org/pep-0492/)