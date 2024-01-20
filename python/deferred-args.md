# Deferred values and default args

This is an alternative (or complement) to [PEP 671](https://peps.python.org/pep-0671/) and [defer expressions](https://discuss.python.org/t/defer-expressions/43490). Some notes:

* This does NOT contain new syntax!
   * but it doesn't forbid adding syntax sugar later to improve usability of this feature
* This DOES contain changes to the data model
* This contains a (backwards compatible) change to function default semantics
* This suggest a new stdlib module (although it could really be provided as a 3rd-party library, which could actually have some use on older python versions)

## Deferred objects

A deferred object is any object with a `__undefer__` method. It's not a defined type, it's just a protocol. In fact, for static typing purposes, you could define:

```python
class Deferred[T](Protocol):
    def __undefer__(self) -> T: ...
```

Note that this change is backwards compatible, because no object in reasonable existing python code has an `__undefer__` method, being a previously unspecified special method;
in general we don't care about breaking compatibility with code that uses "invalid" special method names.

## The `deferred` module

The following new stdlib module could be implemented in Python and has some useful functions (that will be used in examples/tests below)

```python
import copy

def undefer[T](value: Deferred[T] | T) -> T:
    undefer_slot = getattr(type(value), "__undefer__", None)
    return value if undefer_slot is None else undefer_slot(value)

class Call[T]:
    """Deferred value that resolves by calling a argumentless function"""
    def __init__(self, callable: Callable[[], T]) -> None:
        self.callable = callable

    def __undefer__(self) -> T:
        return self.callable()

class Copy[T]:
    """Deferred value that resolves by deep copying a value"""
    def __init__(self, value: T) -> None:
        self.value = value

    def __undefer__(self) -> T:
        return copy.deepcopy(self.value)

class List[T]:
    """Deferred value that resolves to a new list of specified elements"""
    def __init__(self, *values: T) -> None:
        self.values = values

    def __undefer__(self) -> list[T]:
        return list(self.values)
```

## Use in default args

The only semantic change on runtime is when starting a function call and filling missing values with defaults (in 3.12 that's the final section of `ceval.c::initalize_locals`).
There, after filling each missing argument with a default `d`, if `d` has an `__undefer__` method, then the default gets `d.__undefer__()` instead. So for example these tests should pass:

```python
def example(arg=deferred.Call(datetime.now)):
    return arg

assert example(123) == 123
t1 = example()
time.sleep(1)
t2 = example()
assert type(t1) is datetime
assert type(t2) is datetime
assert t1 < t2
```

Note that the implicit `__undefer__` call *only* happens when filling in default values. So the following test should pass:

```python
def fail():
    raise NotImplementedError  # This should never be called

def_value = Call(fail)
result = example(def_value)
assert result is def_value  # no call to __undefer__, argument for example was provided

def example_2(arg=def_value):
    pass

example_2(123) # Does nothing

with assertRaises(NotImplementedError):
    example_2()  # Tries to initialize the missing argument, calls `__undefer__` which calls `fail()`
```

Evaluation of deferred arguments is done in left-to-right order (based on the ordering in the function definition)


## Example use cases

```python
# default that is dynamic (you want the call-time value)
def log_event(event, timestamp=deferred.Call(datetime.now)):
    print(f"{timestamp}: {event}")

# Mutable default, recreated each time. It would be a bug to use an undeferred in __init__
class Palette
    def __init__(self, colors=deferred.List("red", "blue")):
        self.colors = colors
    def add_color(self, new_color):
        self.colors.append(new_color)

# Note that deferred.List(), deferred.Call(list), and deferred.Copy([]) behauve roughly the same way for empty lists, which is a common case
def add_item(item, target=deferred.Copy([])):
    target.append(item)
    return target
```

## Use in dataclasses and other places beyond function defaults

A library like dataclasses could choose to support deferreds, so you could define:

```python
@dataclass
class Palette:
    colors = deferred.List("red", "blue")
```

essentially, the implementation of the library needs to call `deferred.undefer(x)` when pulling a value that **could** be deferred. the same could be done in namedtuple, and in third
party libraries that need similar functionality. Libraries that want to provide early support for deferred values in user data (I imagine some config data structures in Django or FastAPI
could benefit from this, for example) could do it by providing their own copy of the "deferred.undefer" implementation,
without breaking backwards compatibility, and allow users of python 3.13 (or whenever this is implemented) to take advantage of deferred objects.

## Relation with PEP 671

This is a "smaller" proposal than PEP 671: 
* no syntax change
* very lightweight semantic change (and its implementation)
* no change to the compiler or VM
* less memory impact (adds 1 pointer to each type rather than 2 to each function).
It covers several scenarios that PEP 671 considers out of scope. 

Obviously, the syntax is bulkier. But the proposals are actually not "incompatible", and if we're moving towards PEP-671 I think this idea could be used as the "backend implementation" of the PEP,
by building small "undefer" objects for every `x => expr` default argument.

FIXME: it would be reasonably easy to build a code object for evaluating the expresion, but getting the closures properly set inside the function that hasn't start running may require some extra magic. That is only needed if we want the "access other arguments" behaviour

See limitations below for some details about issues that this proposal doesn't cover.

## Relation with Cornelius Krupp's proposal

This is a very similar proposal, but again I've gotten rid of the special syntax. What Cornelius calls "defer <expression>" I write as deferred.Call(lambda: <expression>). His semantics for function calls is the same,
and the usage elsewhere is explicit. I intentionally removed the callability with `()` to avoid mix-up with lambdas, and have an explicit `undefer()` (which doesn't need special syntax, it's just a library call)

The ability to define alternate forms of `__undefer__` allow to implement other behaviours that go beyond calling a faction (which is general, but it's nice to be able to specify alternatives). In this sense, you can get results that are more similar to the [Late](https://pypi.org/project/Late/) library, for example providing copying behaviour.

## Isn't this just a lambda by other name?

Well... yes. That's the whole point. A deferred is an argument-less lambda, with an "entry point" named `__undefer__` instead of `__call__`. That rename is what prevents any confusion where you may actually want the function rather than its result: Using lambdas for deferred evaluation is ambiguous. Do you want the function itself or its result?
Given that we can assume that no reasonable existing code uses `__undefer__`, and no new code uses `__undefer__` except for the use cases defined here, the ambiguity is removed. A deferred is unambiguosly desired to be replaced whenever its a default ina missing argument, or when someone calls explicitly `undefer()`.

## Limitations

This does not cover the `bisect_right(a, x, lo=0, hi=>len(a), *, key=None)`, described in PEP 671, because the undefer hook doesn't have access to other arguments.

This technically could be fixed by passing the rest of the argument functions to `__undefer__`, which would require making `__undefer__` typically fully
variadic (i.e. `__undefer__(self, *args, **kwargs)`). I didn't add this to the proposal to keep it simple and because I think the complexity is not worth it for the few extra cases covered,
but if there's demand for supporting those cases I wouldn't mind exploring an extension of the proposal.

PEP 671 mentions problems in the status quo about not having proper metadata for functions using None/sentinels (it doesn't describe how PEP671 solves it, but I assume that the help generation); much of this could be solved again by providing reasonable `__repr__` implementations for the the library deferred objects.

## A note for typecheckers

Type-checkers should probably have a special rule regarding default arguments where a `Deferred[T]` is an allowed default value for an argument of type `T`. Other than that rules are unchanged.

For example users would declare:
```
def log_event(event: str, timestamp: datetime = deferred.Call(datetime.now)):
    print(f"{timestamp}: {event}")
```
