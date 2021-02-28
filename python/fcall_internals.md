# Function call internals in python

These are my notes trying to document what goes on from a function call like "f(a,b,x=44)" until the body a function "def f(aa, ab, ac=10, x=20)" gets executed.
Note that some of these things are simplified for functions without keywords, I'm covering the most complicated case.

Some of this was in https://eli.thegreenplace.net/2012/03/23/python-internals-how-callables-work, but it was outdated. I'm writing this for Python 3.9.
Something else this covers is how the new frame gets built based on the arguments passed.

## Before the call

The caller places the function in the stack

```
  1           0 LOAD_NAME                0 (f)
```

And on top of that, positional arguments are passed

```
              2 LOAD_NAME                1 (a)
              4 LOAD_NAME                2 (b)
```

Finally, all keyword arguments are passed, followed by a (const, precomputed) tuple with the keyword names

```
              6 LOAD_CONST               0 (44)
              8 LOAD_CONST               1 (('x',))
```

At this point the top of the stack has: [<function>, <arg1>, ..., <argn>, <kwarg1>, ..., <kwargm>, (<kwname1>, ..., <kwnamem>)]

## The call in the VM

The opcode that triggers the call is 

```
             10 CALL_FUNCTION_KW         3
```

The argument 3 specifies how many arguments (either positional or keyword) were passed. At this point the interpreter is still far from knowing the signature of f
(in fact it hasn't even check that the <function> object placed in the stack is callable at all.

The opcode is implented [here](https://github.com/python/cpython/blob/3.9/Python/ceval.c#L3527); but all that does is delegate into `call_function()`.

For calls with no keyword, a slightly different `CALL_FUNCTION` is used.

## `call_function` (ceval.c)

Defined [here](https://github.com/python/cpython/blob/3.9/Python/ceval.c#L5059)

This function receives:
* thread state
* A reference to the stack pointer (the extra indirection allows this function to mutate it)
* the total number of arguments (3 in our example, the argment to CALL_FUNCTION_KW)
* the tuple with the keyword argument names, which is  `("x",)` in our example

It's main job is to "parse" the stack, and call `PyObject_Vectorcall()` (and converting data to pass it there). It also has some extra code related to tracing and stack unwinding after the call.

## `PyObject_Vectorcall` and `_PyObject_VectorcallTstate` (abstract.h)

Defined [here](https://github.com/python/cpython/blob/3.9/Include/cpython/abstract.h#L122); this is a small inline (not that it lives in a .h)

It receives:
* The function/callable object
* a pointer to the stack area starting at the first argument (<argument1>)
* the number of **positional** arguments. This value may be bitwise or'd with `PY_VECTORCALL_ARGUMENTS_OFFSET` which is a bit
  mask that allows the function to update the stack value with the return. In our case this will be `2|PY_VECTORCALL_ARGUMENTS_OFFSET`
* the tuple with the keyword argument names, which is `("x",)` in our example

It looks up the thread state, and passes it and all the data above unchanged to `_PyObject_VectorcallTstate`, defined [right before](https://github.com/python/cpython/blob/3.9/Include/cpython/abstract.h#L84)

Note that this function doesn't care much above the stack pointer being actually stack, this is just a fast way of having all the argument
values within a C array (which is the underlying representation for the stack).

What this function does is:

* It calls `func = PyVectorcall_Function(callable)`, which essentially gets type(callable) and check if it has the bitflag `Py_TPFLAGS_HAVE_VECTORCALL`.
  * If it does, PyVectorcall_Function looks up the `tp_vector_callable_offset`, and based on it returns a "vector call function" buried within the callable at the given offset.
    * Then, `_PyObject_VectorcallTstate` calls that function passing the callable, argument vector, positional count, and names tuple
  * If it doesn't PyVectorcall_Function returns NULL
    * Then, `_PyObject_VectorcallTstate` calls `_PyObject_MakeTpCall` passing the callable, argument vector, positional count (clearing up the `PY_VECTORCALL_ARGUMENTS_OFFSET` flag) , and names tuple

As you note, there are two different scenarios, depedning if the callable type supports this "vectorcall". This protocol comes
from [PEP-590](https://www.python.org/dev/peps/pep-0590/), and provides a fast way to implement many function calls.
The `MakeTpCall` API is more general but less efficient. In this case, we're interested in call into Python functions, which support this vectorcall protocol,
so we'll only follow that arm.

## Into the function object

At this point, the behaviour depends on a function pointer pulled from the callable, so behaviour could be very different depending on what we're calling
(a function, a custom object with `__call__`, a builtin?). I'll follow the path for a regular python function created with a `def` statement.

The function objects (in C) have a `vectorcall` attribute always set to `_PyFunction_Vectorcall`. The function type sets the
`tp_vector_callable_offset` to the offset of that attribute to allow `_PyObject_VectorcallTstate` to find it. So `_PyObject_VectorcallTstate` will in our
example end up calling `_PyFunction_Vectorcall`.

This is where the process got farther away from the interpreter loop, and at this point we start to walk back into the evaluation loop.

## `_PyFunction_Vectorcall` (Objects/call.c)

Defined [here](https://github.com/python/cpython/blob/3.9/Objects/call.c#L345)

It receives:
* The function/callable object
* a pointer to the stack area starting at the first argument (<argument1>)
* the number of **positional** arguments. This value may be bitwise or'd with `PY_VECTORCALL_ARGUMENTS_OFFSET` which is a bit
  mask that allows the function to update the stack value with the return. In our case this will be `2|PY_VECTORCALL_ARGUMENTS_OFFSET`
* the tuple with the keyword argument names, which is `("x",)` in our example

This function does some more work on its arguments, with the goal of finally calling `_PyEval_EvalCode`. This includes:

* Collecting some data that will be required to enter the evaluation loop:
  * The thread state
  * The code object (from the function)
  * The global namespace (from the function)
  * The function defaults (from the function)
* At this point, there's a gain an optimization shortcut. If the call has no keyword arguments, and the function has certain properties 
  (no closure variables, doesn't have keyword-only parameters, etc), a `function_code_fastcall` is attempted.
* Otherwise, more information is collected:
  * the function name and qualified name
  * the closure
  * default for keyword-only arguments
* And the `_PyEval_EvalCode` function is called

## `function_code_fastcall` (Objects/call.c)

Defined[here](https://github.com/python/cpython/blob/3.9/Objects/call.c#L307)

Not all function calls go through here (in particular, this doesn't support keyword args)

## `_PyEval_EvalCode` (ceval.c)

defined [here](https://github.com/python/cpython/blob/3.9/Python/ceval.c#L4069)

## `PyEval_EvalFrame`

defined [here](https://github.com/python/cpython/blob/3.9/Python/ceval.c#L838)

Through a series of thin wrappers (`_PyEval_EvalFrame -> _PyEval_EvalFrameDefault`), you end up in the main interpreter loop, running the function code. Welcome back!

## Questions

* Why in the path `call_function -> PyObject_VectorCall -> _PyObject_VectorcallTstate` the thread state is lost and recovered? wouldn't it be better to `call_function -> _PyObject_VectorcallTstate`
