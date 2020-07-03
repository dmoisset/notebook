# Semantics of the `match` statement in python

This notes will try to accurately describe the semantics of the pattern matching statement as described in [PEP-622](https://www.python.org/dev/peps/pep-0622/). It does it through pseudocode which mostly looks like accurate python, and "translates" the new constructs to pre-PEP python. This does NOT meen that what I'm showing is the actual implementation. The main goal of this is descriptive, and following this should help you understand the abstract behaviour.

## Basic definitions

For each syntact **pattern** construct _P_ we will say that the semantics of _P_, or _\<P\>_ is a function that takes any object as an input and returns either a special value `Fail` on **failure** or a binding in case of a **match**. A **binding** is dictionary with identifiers as keys, and arbitrary objects as values, and it represents which objects will be assigned to each local name by the pattern match. For example, a pattern `[a, 2]` matched with the value `(1, 2)`, will return the binding `{"a": 1}`; if matched with the tuple `(2, 1)` it will return `Fail`. Note that a successful match can still return an empty binding `{}`
which is not equivalent to a `Fail` result.

We will also call _FV(P)_ to the set of free variables of the pattern. In practice that mean all the names that appear in capture patterns and to the left of walrus patterns. Note that this set is known at compile time

## Semantic of a simple `match` statement

Let's say we have patterns P₁, P₂, ..., Pᵢ with i≥1, and statements (technically [suites](https://docs.python.org/3/reference/compound_stmts.html#grammar-token-suite)) S₁, S₂, ..., Sᵢ , an expression 𝐸 in the follow match statement:

```python
match 𝐸:
    case P₁: S₁
    case P₂: S₂
    ...
    case Pᵢ: Sᵢ
```

Then it could be translated to the following pseudo-python statement while keeping the meaning:

```python
_tmp = E
if (_m := <P₁>(_tmp) ) != Fail:  # Check if there's a match
    for each name 𝑁 in FV(P₁):
        𝑁 = _m["𝑁"]  # Bind name present in the pattern
    S₁  # Run the clause
elif (_m := <P₂>(_tmp) ) != Fail:
    for each name 𝑁 in FV(P₂):
        𝑁 = _m["𝑁"]
    S₂
elif ...
elif (_m := <Pᵢ>(_tmp) ) != Fail:
    for each name 𝑁 in FV(Pᵢ):
        𝑁 = _m["𝑁"]
    Sᵢ
```

During this documents, variables prefixed with `_` will be considered auxiliary internal variables to describe the semantic, and do not represent actual names being modified.

Note that the pseudo "for each" loop described above depends on information available on compile time and could be unrolled if you want a pure python translation.

## Semantic of a guarded `match` statement

Each case clause can have a guard at the end. These clauses are an arbitrary expression. Clauses have access to the variables bound by the match. For simplicity,
let's assume that in a match statement all `case` clauses have a guard (this can be achieved by adding `if True` for those clauses without a guard). Let's extend the previous pattern with guards G₁, G₂, ..., Gᵢ, each of those an expression:

```python
match 𝐸:
    case P₁ if G₁: S₁
    case P₂ if G₂: S₂
    ...
    case Pᵢ if Gᵢ: Sᵢ
```

Then it could be translated to the following pseudo-python statement while keeping the meaning, generalizing the semantics above:

```python
_tmp = E
for each k in 1, 2, ... i:
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

### Wildcard Pattern

### Capture Pattern

### Sequence Pattern

### Mapping Pattern

### Class Pattern

## Caveats about variable binding

