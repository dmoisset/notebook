# Semantics of the `match` statement in Python

This notes will try to accurately describe the semantics of the pattern matching statement proposed in [PEP-622](https://www.python.org/dev/peps/pep-0622/). It does it through pseudocode which mostly looks like accurate python, and "translates" the new constructs to pre-PEP python. This does NOT meen that what I'm showing is the actual implementation. The main goal of this is descriptive, and following this should help you understand the abstract behaviour.

## Basic definitions

For each syntact **pattern** construct _P_ we will say that the semantics of _P_, or _\<P\>_ is a function that takes any object as an input and returns either a special value `Fail` on **failure** or a binding in case of a **match**. A **binding** is dictionary with identifiers as keys, and arbitrary objects as values, and it represents which objects will be assigned to each local name by the pattern match. For example, a pattern `[a, 2]` matched with the value `(1, 2)`, will return the binding `{"a": 1}`; if matched with the tuple `(2, 1)` it will return `Fail`. Note that a successful match can still return an empty binding `{}`
which is not equivalent to a `Fail` result.

We will also call _FV(P)_ to the set of free variables of the pattern. In practice that mean all the names that appear in capture patterns and to the left of walrus patterns. Note that this set is known at compile time

## Semantic of a simple `match` statement

Let's say we have patterns P₁, P₂, ..., Pₙ with n≥1, and statements (technically [suites](https://docs.python.org/3/reference/compound_stmts.html#grammar-token-suite)) S₁, S₂, ..., Sₙ , an expression 𝐸 in the follow match statement:

```python
match 𝐸:
    case P₁: S₁
    case P₂: S₂
    ...
    case Pₙ: Sₙ
```

Then it could be translated to the following pseudo-python statement while keeping the meaning:

```python
_tmp = 𝐸
if (_m := <P₁>(_tmp) ) != Fail:  # Check if there's a match
    for each name 𝑁 in FV(P₁):
        𝑁 = _m["𝑁"]  # Bind name present in the pattern
    S₁  # Run the clause
elif (_m := <P₂>(_tmp) ) != Fail:
    for each name 𝑁 in FV(P₂):
        𝑁 = _m["𝑁"]
    S₂
elif ...
elif (_m := <Pₙ>(_tmp) ) != Fail:
    for each name 𝑁 in FV(Pₙ):
        𝑁 = _m["𝑁"]
    Sₙ
```

During this documents, variables prefixed with `_` will be considered auxiliary internal variables to describe the semantic, and do not represent actual names being modified.

Note that the pseudo "for each" loop described above depends on information available on compile time and could be unrolled if you want a pure python translation.

## Semantic of a guarded `match` statement

Each case clause can have a guard at the end. These clauses are an arbitrary expression. Clauses have access to the variables bound by the match. For simplicity,
let's assume that in a match statement all `case` clauses have a guard (this can be achieved by adding `if True` for those clauses without a guard). Let's extend the previous pattern with guards G₁, G₂, ..., Gₙ, each of those an expression:

```python
match 𝐸:
    case P₁ if G₁: S₁
    case P₂ if G₂: S₂
    ...
    case Pₙ if Gₙ: Sₙ
```

Then it could be translated to the following pseudo-python statement while keeping the meaning, generalizing the semantics above:

```python
_tmp = 𝐸
for each k in 1, 2, ... n:
    if (_m := <Pₖ>(_tmp) ) != Fail:  # Check if there's a match
        for each name 𝑁 in FV(Pₖ):
            𝑁 = _m["𝑁"]  # Bind name present in the pattern
        if Gₖ:
            Sₖ  # Run the clause
            goto DONE
DONE: 
```

## Semantics of each pattern type

### Literal Pattern

A literal pattern consists of a literal 𝐿. Its semantics correspond to this function

```python
def literal(o: object) -> Union[Binding, Fail]:
    return {} if o == 𝐿 else Fail
```

Also, FV(𝐿) = ∅ (the empty set)

### Constant Pattern

A literal pattern consists of an expression 𝐸 (restricted syntactically to a very specific set of expressions). Its semantics correspond to this function

```python
def constant(o: object) -> Union[Binding, Fail]:
    return {} if o == 𝐸 else Fail
```

Also, FV(𝐸) = ∅

### Wildcard Pattern

Syntatically there is no such thing as a wildcard pattern, but treating the `_` pattern as a special case make this description simpler. The semantics of the wildcard pattern is always

```python
def wildcard(o: object) -> Binding:
    return {}
```

Also, FV(`_`) = ∅

### Capture Pattern

A capture pattern consists of a single name 𝑁. Its semantics are:

```python
def capture(o: object) -> Binding:
    return {"𝑁": o}
```

Also, FV(𝑁) = {𝑁}

### Sequence Pattern

Sequence patterns can be "simple" (no `*x` subpattern), or "extended" if it has a `*x` in it. Let's see first the semantics of a simple pattern, which has the form [P₁, P₂, ..., Pₙ] with 𝑛≥0, where each Pᵢ is a pattern. Its semantics are:

```python
def capture(o: object) -> Union[Fail, Binding]:
    if isinstance(o, (str, bytes, bytearray)): return Fail  # These types are forbidden
    if not isinstance(o, collections.abc.Sequence): return Fail
    if len(o) != 𝑛: return Fail
    binding = {}
    for each i in 1, 2, ..., 𝑛:
        elem_binding = <Pᵢ>(o[i-1])
        if elem_binding == Fail: return Fail
        binding.update(elem_binding)
    return binding
```

We define FV([P₁, P₂, ..., Pₙ]) = FV(P₁) ∪ FV(P₂) ∪ ... ∪ FV(Pₙ)

Extended matches will be of the form [L₁, L₂, ..., Lₙ, `*`𝑁, R₁, R₂, ..., Rₘ], where 𝑛≥0, 𝑚≥0, and Lᵢ and Rⱼ are patterns. Its semantics are:

```python
def capture(o: object) -> Union[Fail, Binding]:
    if isinstance(o, (str, bytes, bytearray)): return Fail  # These types are forbidden
    if not isinstance(o, collections.abc.Sequence): return Fail
    if len(o) < 𝑛+𝑚: return Fail
    binding = {}
    # Bind the left
    for each i in 1, 2, ..., 𝑛:
        elem_binding = <Pᵢ>(o[i-1])
        if elem_binding == Fail: return Fail
        binding.update(elem_binding)
    # Bind the right
    for each i in 1, 2, ..., 𝑚:
        elem_binding = <Pᵢ>(o[-𝑚+(i-1)])
        if elem_binding == Fail: return Fail
        binding.update(elem_binding)
    # bind the middle
    middle = list(o[𝑛:len(o)-𝑚])
    binding.update(<𝑁>(middle))
    return binding
```


### Mapping Pattern

### Class Pattern

## Caveats about variable binding

