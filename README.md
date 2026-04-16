---
tags:
  - python
  - internals
  - cpython
---

# Python Internals

A reference on how Python works under the hood — object model, namespaces, memory management, CPython implementation details, and the data model.

---

## Table of Contents

- [Python and Objects](#1-python-and-objects)
  - [Everything is an Object](#11-everything-in-python-is-an-object)
  - [CPython: PyObject Under the Hood](#12-cpython-pyobject-under-the-hood)
  - [Objects and Attributes](#13-objects-and-attributes)
  - [Types and `type()`](#14-types-with-type)
  - [Method Resolution Order (MRO)](#15-method-resolution-order-mro)
  - [Callables](#16-callables)
  - [Object Size](#17-object-size)
- [Names and Namespaces](#2-names-and-namespaces)
  - [Names and Namespaces](#21-names-and-namespaces)
  - [`id()`, `is`, and `==`](#22-id-is-and-)
  - [Object Attributes and Namespaces](#23-object-attributes-and-namespaces)
  - [Custom Namespaces](#24-creating-a-custom-namespace)
  - [Reference Counting](#25-object-reference-count)
  - [Integer Caching and String Interning](#26-integer-caching-and-string-interning)
  - [Ways to Set Names in a Namespace](#27-various-methods-to-set-names-in-a-namespace)
  - [Overwriting Builtin Names](#28-overwriting-builtin-names)
  - [Scopes and the LEGB Rule](#29-scopes-and-the-legb-rule)
  - [`global` and `nonlocal`](#210-global-and-nonlocal)
  - [`locals()` and `globals()`](#211-locals-and-globals)
  - [The `import` Statement](#212-the-import-statement)
  - [Functions and Namespaces](#213-functions-and-namespaces)
- [Memory Management and Garbage Collection](#3-memory-management-and-garbage-collection)
  - [Reference Counting in CPython](#31-reference-counting-in-cpython)
  - [Cycle Detection](#32-cycle-detection)
  - [Generational GC](#33-generational-garbage-collection)
  - [Weak References](#34-weak-references)
- [Bytecode and the CPython VM](#4-bytecode-and-the-cpython-vm)
  - [Compilation Pipeline](#41-compilation-pipeline)
  - [Inspecting Bytecode with `dis`](#42-inspecting-bytecode-with-dis)
  - [Code Objects](#43-code-objects)
  - [The GIL](#44-the-global-interpreter-lock-gil)
- [The Descriptor Protocol](#5-the-descriptor-protocol)
  - [How Attribute Lookup Works](#51-how-attribute-lookup-works)
  - [Data vs Non-data Descriptors](#52-data-vs-non-data-descriptors)
  - [`property` as a Descriptor](#53-property-as-a-descriptor)
  - [`__slots__`](#54-__slots__)
- [Metaclasses](#6-metaclasses)
  - [`type` is the Metaclass of All Classes](#61-type-is-the-metaclass-of-all-classes)
  - [`__new__` vs `__init__`](#62-__new__-vs-__init__)
  - [Custom Metaclasses](#63-custom-metaclasses)

---

## 1. Python and Objects

### 1.1. Everything in Python is an Object

All values in Python — integers, strings, functions, classes, modules — are objects in memory. A name (variable) is just a reference to an object; it is not the object itself.

When you write `v = 1` at the REPL:

1. The interpreter determines the appropriate built-in type for the value (`int` here).
2. An instance of that type is created in the heap — a block of memory with metadata and the value.
3. The instance inherits attributes from its type class.
4. A pointer named `v` is created in the current namespace, pointing to that object.

Every object has:

| Property | Description |
| --- | --- |
| **Type** | The class it was instantiated from (`int`, `str`, a custom class, etc.) |
| **Value** | The payload (e.g. `42`, `"hello"`) |
| **Identity** | A unique integer ID for the lifetime of the object (`id()`) |
| **Reference count** | How many names/containers currently point to it |
| **Attributes** | Methods and data inherited from its type, plus any instance-specific data |

---

### 1.2. CPython: PyObject Under the Hood

In CPython (the reference implementation), every Python object is represented by a C struct. The base struct is `PyObject`, defined in `Include/object.h`:

```c
typedef struct _object {
    Py_ssize_t ob_refcnt;   /* reference count */
    PyTypeObject *ob_type;  /* pointer to the type object */
} PyObject;
```

For variable-length objects (lists, strings) there is a slightly extended variant:

```c
typedef struct {
    PyObject ob_base;
    Py_ssize_t ob_size;   /* number of items in the collection */
} PyVarObject;
```

Key points:

- `ob_refcnt` drives the **reference counting** garbage collector. Every time a reference is added (assignment, passing to a function, appending to a list), `ob_refcnt` is incremented. When a reference is dropped, it is decremented. When it hits zero, the object is deallocated immediately.
- `ob_type` is a pointer to a `PyTypeObject` struct that describes the type — its name, its size, its slots for `__add__`, `__repr__`, `__hash__`, etc. When you call a method on an object, CPython follows `ob_type` to find the implementation.
- `id(obj)` returns the memory address of the underlying `PyObject` (on CPython). It is unique only for the lifetime of the object.

The implication: **there is no "primitive" in Python**. Even the integer `1` is a heap-allocated `PyLongObject` with a refcount and a type pointer. This is why Python is slower than C for arithmetic — every operation involves pointer dereferences.

---

### 1.3. Objects and Attributes

Object attributes are inherited from the class that instantiated the object. They live in the namespace established by that class (and its MRO).

```python
a = "test"
type(a)       # str
a.__class__   # str
dir(a)        # lists all attributes: __add__, __contains__, upper, lower, ...
```

Attributes are stored in a dictionary (`__dict__`) for most objects. Calling `a.upper()` is roughly equivalent to `type(a).__dict__['upper'].__get__(a, type(a))()` — an attribute lookup that goes through the descriptor protocol (see [Section 5](#5-the-descriptor-protocol)).

---

### 1.4. Types with `type()`

`type()` returns the type of an object. Internally it reads `ob_type` from the `PyObject` struct.

```python
type(1)       # int
type(int)     # type
type(type)    # type   ← type is its own metaclass
```

`type()` is also a constructor for creating new types dynamically:

```python
# type(name, bases, dict) creates a new class
MyClass = type('MyClass', (object,), {'x': 42, 'greet': lambda self: 'hello'})
MyClass().greet()   # 'hello'
```

The chain ends here: `type(object)` is `type`, and `type(type)` is `type`.

```text
int       → type      → type  (metaclass of all classes)
object    → type
type      → type
```

```python
# Equivalent: calling type() reads __class__
type(1)       # int
(1).__class__ # int
```

---

### 1.5. Method Resolution Order (MRO)

MRO is the order in which Python searches for a method or attribute when looking it up on an object.

CPython uses the **C3 linearization** algorithm (since Python 2.3). For single inheritance it is trivial; for multiple inheritance it ensures:

- A class always comes before its parents.
- The relative order of parent classes is preserved.
- The MRO is consistent (no class appears twice).

```python
import inspect

inspect.getmro(bool)   # (bool, int, object)
inspect.getmro(int)    # (int, object)

# For a more complex case
class A: pass
class B(A): pass
class C(A): pass
class D(B, C): pass

D.__mro__
# (<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)
```

`__mro__` is available on **classes**, not instances:

```python
a = 1
a.__mro__       # AttributeError: 'int' object has no attribute '__mro__'
int.__mro__     # (int, object)
type(a).__mro__ # (int, object)
```

**MRO and inheritance walkthrough:**

```python
True.__class__                    # bool
True.__class__.__bases__          # (int,)   ← bool inherits from int
True.__class__.__bases__[0]       # int
True.__class__.__bases__[0].__bases__  # (object,)
```

All inheritance chains ultimately end at `object`. Calling `object.__bases__` returns `()` — the empty tuple.

---

### 1.6. Callables

An object is callable if calling it (`obj()`) makes sense — i.e., it implements `__call__`. The built-in `callable()` checks for this.

```python
x = int(1212.3)
callable(x)          # False — int instances are not callable

callable(int)        # True  — the int class itself is callable (creates instances)
callable(len)        # True  — built-in function
callable(lambda: 1)  # True  — lambda

class MyClass:
    pass

callable(MyClass)    # True — classes are callable

class WithCall:
    def __call__(self):
        return 42

obj = WithCall()
callable(obj)        # True — instance with __call__
obj()                # 42
```

Any class that defines `__call__` produces callable instances. This is how decorators implemented as classes work.

---

### 1.7. Object Size

`sys.getsizeof()` returns the size of an object in bytes — specifically, the size of the object itself, **not** the objects it refers to.

```python
import sys

sys.getsizeof(1)           # 28   — small int in CPython
sys.getsizeof(2**100)      # 40   — bignum uses more words
sys.getsizeof("hello")     # 54   — base string overhead + 5 chars
sys.getsizeof([])          # 56   — empty list
sys.getsizeof([1, 2, 3])   # 88   — list with 3 references (not the ints themselves)
sys.getsizeof({})          # 64   — empty dict
```

For containers, use `tracemalloc` or `pympler.asizeof()` to get the total recursive size including referenced objects.

```python
import tracemalloc
tracemalloc.start()
x = [list(range(1000)) for _ in range(100)]
snapshot = tracemalloc.take_snapshot()
for stat in snapshot.statistics('lineno')[:3]:
    print(stat)
```

---

## 2. Names and Namespaces

### 2.1. Names and Namespaces

A **name** (variable) is a reference to an object in memory — a label, not a container. The same object can have multiple names; the same name cannot point to multiple objects simultaneously.

A **namespace** is a mapping of names to object references — conceptually a dictionary. Different namespaces are isolated from each other: two namespaces can have a name `x` pointing to different objects.

A **scope** is the region of code where a namespace is directly accessible without dot notation.

Key points:

1. Assignment (`=`), `del`, and `import` are namespace operations — they modify the name-to-object mapping, not the objects themselves.
2. Objects do not have types; names do not have types. **Objects** have types.
3. A name that refers to an `int` today can refer to a `str` tomorrow — the name has no type constraint.
4. Deleting a name (`del x`) removes the mapping entry; it does not destroy the object. The object is destroyed only when its reference count reaches zero.
5. Dot notation (`.`) accesses a sub-namespace: `sys.version` means "look up `version` in the namespace of the object `sys`."

```python
a = 300    # create int object 300, bind name 'a' to it
b = a      # bind name 'b' to the SAME object (not a copy)
id(a) == id(b)  # True — same object

a = 400    # create new int object 400, rebind 'a' — 'b' still points to 300
id(a) == id(b)  # False — now different objects
b          # 300
```

`dir()` lists names in the current namespace scope:

```python
x = 42
dir()      # [..., 'x', ...]
```

---

### 2.2. `id()`, `is`, and `==`

| Operator/Function | What it checks |
| --- | --- |
| `id(obj)` | Returns the memory address of the object (on CPython) |
| `a is b` | True if `a` and `b` refer to the **same object** (same `id`) |
| `a == b` | True if `a` and `b` have the **same value** (calls `__eq__`) |

```python
a = 400
b = a
id(a) == id(b)   # True
a is b           # True  — same object
a == b           # True  — same value

b = 400          # new int object (outside the small-int cache)
id(a) == id(b)   # False on CPython (two separate int objects)
a is b           # False
a == b           # True  — same value, different objects

del b
# b is gone from namespace; a still points to the object
del a
# object has no more references → garbage collected
```

**Rule:** Use `==` to compare values. Use `is` only when checking identity (e.g. `if x is None`, `if x is not None`) — this is the standard Python style because `None`, `True`, and `False` are singletons.

**A single name cannot refer to multiple objects simultaneously:**

```python
a = 10
id(a)      # address of int(10)
a = "Walking"
id(a)      # different address — a now points to a str object
```

---

### 2.3. Object Attributes and Namespaces

Object attributes form a namespace specific to that object. They are accessed via dot notation and stored in the object's `__dict__` (unless `__slots__` is used — see [Section 5.4](#54-__slots__)).

```python
a = "test"
a.__dict__    # AttributeError: str has no __dict__ (it uses C-level slots)

class Foo:
    def __init__(self):
        self.x = 1
        self.y = 2

f = Foo()
f.__dict__    # {'x': 1, 'y': 2}
```

For built-in types, the attribute namespace lives in the `PyTypeObject` C struct rather than a Python dict — which is why `str.__dict__` exists but `"hello".__dict__` does not.

---

### 2.4. Creating a Custom Namespace

`types.SimpleNamespace` (Python 3.3+) provides a clean, attribute-based namespace:

```python
from types import SimpleNamespace

ns = SimpleNamespace()
ns.host = "localhost"
ns.port = 8080

ns               # namespace(host='localhost', port=8080)
ns.host          # 'localhost'
ns.__dict__      # {'host': 'localhost', 'port': 8080}
```

It is equivalent to defining a bare class:

```python
class NS:
    pass

ns = NS()
ns.host = "localhost"
```

`SimpleNamespace` also supports equality comparison (two instances with the same attributes are equal) and a useful `repr`.

---

### 2.5. Object Reference Count

`sys.getrefcount()` returns the number of references to an object. Note: the call itself adds a temporary reference, so the count is always at least 1 higher than the "true" count.

```python
from sys import getrefcount

a = []
getrefcount(a)      # 2  (one for 'a', one for the getrefcount argument)

b = a
getrefcount(a)      # 3  (one more: 'b' also refers to the list)

del b
getrefcount(a)      # 2  (back to 'a' + getrefcount temp)
```

---

### 2.6. Integer Caching and String Interning

CPython implements two important optimizations that affect object identity.

#### Integer caching (small-int cache)

CPython pre-allocates integer objects for values in the range **-5 to 256**. These objects are singletons — every reference to the integer `5` in a CPython process points to the *same* object.

```python
a = 100
b = 100
a is b      # True  — same cached object

a = 300
b = 300
a is b      # False — outside the cache; two distinct objects
a == b      # True  — same value

# Confirming the cache boundary
(256).__class__    # int
x, y = 256, 256
x is y    # True

x, y = 257, 257
x is y    # False (CPython default; may vary in interactive sessions)
```

The cache is populated at interpreter start-up. Frequently used integers (0, 1, 2, …) have very high reference counts because the interpreter itself holds references.

```python
[(i, getrefcount(i)) for i in range(5)]
# [(0, ~2000), (1, ~1800), (2, ~600), ...]  — high counts from interpreter internals
```

#### String interning

CPython automatically **interns** string literals that look like valid identifiers (no spaces, only alphanumeric + `_`). Interned strings are deduplicated — multiple references to the same identifier string point to the same object.

```python
a = "hello"
b = "hello"
a is b      # True  — both interned to the same object

a = "hello world"     # contains a space — not automatically interned
b = "hello world"
a is b      # False (implementation-dependent; often False)

# Force interning manually
import sys
a = sys.intern("hello world")
b = sys.intern("hello world")
a is b      # True
```

**Why it matters:** Interned strings use pointer comparison (`is`) for equality checks, which is O(1) vs O(n) for string comparison. CPython uses interning extensively for attribute name lookups — `obj.method` compares the interned string `"method"` by pointer against the interned keys in `__dict__`.

---

### 2.7. Various Methods to Set Names in a Namespace

#### Direct assignment

```python
a = 1
b = a        # b points to the same object as a
c = b = a    # all three names point to the same object
```

#### Tuple unpacking

```python
a, b, c = 10, 20, 30
a, b, c      # (10, 20, 30)

# Swap without a temporary variable
a, b = b, a  # Python creates the tuple (b, a) first, then unpacks
```

#### Extended iterable unpacking (Python 3)

The `*` operator captures the "rest" of an iterable into a list:

```python
a, b, c, *d = "HelloWorld!"
# a='H', b='e', c='l', d=['l', 'o', 'W', 'o', 'r', 'l', 'd', '!']

a, *b, c = "HelloWorld"
# a='H', b=['e','l','l','o','W','o','r','l'], c='d'

first, *_, last = range(10)
# first=0, last=9, _=[1,2,3,4,5,6,7,8]
```

The starred name always receives a `list`, even when the iterable is a tuple or a string.

#### Importing modules

```python
import os          # binds the name 'os' to the module object
import os as o     # binds to the name 'o'
from os import path          # binds only 'path' into the current namespace
from os import path as p     # same, with alias
```

---

### 2.8. Overwriting Builtin Names

Built-in names (`len`, `list`, `type`, `print`, etc.) live in the `builtins` namespace. Assigning to the same name in the local or global namespace creates a shadow — the builtin is hidden, not destroyed.

```python
a = "hello"
len(a)       # 5

len = "oops"
len(a)       # TypeError: 'str' object is not callable

del len
len(a)       # 5 — builtin is visible again
```

The name resolution order (LEGB, see below) explains why: Python finds `len` in the local/global namespace before reaching builtins. Deleting the shadow exposes the builtin again.

The builtins namespace itself can be found via `__builtins__` in module scope and `builtins` after `import builtins`. You can — but should never — modify it directly:

```python
import builtins
builtins.len = lambda x: 42   # don't do this
```

---

### 2.9. Scopes and the LEGB Rule

Python resolves names using the **LEGB** rule — four nested scopes searched in order:

| Scope | What it is |
| --- | --- |
| **L** — Local | Names defined in the current function |
| **E** — Enclosing | Names in any enclosing functions (closures) |
| **G** — Global | Names at module level |
| **B** — Built-in | Names in the `builtins` module |

The first match wins and the search stops.

```python
x = "global"

def outer():
    x = "enclosing"

    def inner():
        # No local x — finds x in the enclosing scope
        print(x)   # "enclosing"

    inner()

outer()
```

**Local scope and `UnboundLocalError`:**

Python determines at compile time whether a name in a function is local (by checking if it is assigned anywhere in the function body). If a name is assigned anywhere in a function, it is treated as local throughout that function — even before the assignment.

```python
x = "global"

def test():
    print(x)    # UnboundLocalError!
    x = "local" # Because of this assignment, x is local for the entire function

test()
```

Python raises `UnboundLocalError` at the `print(x)` line because `x` is classified as a local variable (due to the assignment below it) but has not yet been assigned when `print` executes.

---

### 2.10. `global` and `nonlocal`

#### `global`

Forces a name to resolve to the global (module-level) scope, even if it is assigned inside the function:

```python
x = "global"

def modifier():
    global x
    print(x)    # "global" — reads from module scope
    x = "modified"

modifier()
print(x)        # "modified" — the module-level x was changed
```

Without `global x`, the assignment `x = "modified"` would create a new local variable and leave the module-level `x` untouched.

#### `nonlocal`

Forces a name to resolve to the nearest **enclosing function scope** (not global). Used in closures and nested functions:

```python
def counter():
    count = 0

    def increment():
        nonlocal count    # refers to 'count' in counter(), not a new local
        count += 1
        return count

    return increment

c = counter()
c()   # 1
c()   # 2
c()   # 3
```

Without `nonlocal count`, `count += 1` would raise `UnboundLocalError` because `count` would be treated as a new local in `increment()`.

**`nonlocal` vs `global`:**

| Keyword | Scope it targets |
| --- | --- |
| `global x` | Module-level (global) scope |
| `nonlocal x` | Nearest enclosing function scope (not global) |

---

### 2.11. `locals()` and `globals()`

Both return dictionaries representing the current namespace, but from different vantage points:

```python
def demo():
    a = 100
    b = 1024
    print("locals:", locals())    # {'a': 100, 'b': 1024}
    print("globals has 'demo':", 'demo' in globals())  # True

demo()
```

- `locals()` — returns the local namespace of the current scope. **Mutating the returned dict does not affect the actual locals** (CPython uses "fast locals" — a C array, not a dict — for function frames; `locals()` copies them into a dict).
- `globals()` — returns the module-level namespace dict. **Mutations here do affect the global namespace.**

```python
x = 10
globals()['x'] = 99
print(x)    # 99 — global namespace was modified
```

**Fast locals caveat:** Inside a function, `locals()['a'] = 500` does not change the local variable `a`, because CPython function frames store locals in a fixed C array (`co_varnames`), not in the dict returned by `locals()`. The dict is a snapshot.

---

### 2.12. The `import` Statement

`import` is a wrapper around the built-in `__import__()` function. It:

1. Checks `sys.modules` — if the module is already loaded, reuse the cached module object (modules are singletons per interpreter).
2. If not cached, finds the source file by searching `sys.path` in order.
3. Compiles the source to bytecode (or loads `.pyc` from `__pycache__/`).
4. Executes the module's top-level code in a new module namespace.
5. Stores the module object in `sys.modules` and binds the name in the current namespace.

```python
import sys
sys.path     # search path for modules
sys.modules  # dict of all loaded modules (name → module object)
```

**Import variants:**

```python
import os                   # binds 'os' in current namespace
import os as operating_sys  # binds 'operating_sys'
from os import path         # binds 'path' directly (no 'os.' prefix needed)
from os import *            # binds all public names from os (avoid in production)
```

**Deleting an import:**

```python
del sys
'sys' in dir()         # False — name removed from namespace
'sys' in sys.modules   # True  — still cached; re-import is instant
```

For programmatic imports, use `importlib`:

```python
import importlib
os = importlib.import_module('os')
```

**Relative imports** (inside packages):

```python
from . import sibling_module       # same package
from .. import parent_module       # parent package
from .utils import helper_func     # specific name from sibling module
```

---

### 2.13. Functions and Namespaces

Defining a function creates a function object in memory and a name for it in the current namespace — exactly the same mechanism as for `int`, `str`, etc.

```python
def func1():
    pass

type(func1)     # <class 'function'>
'func1' in dir()  # True
```

Function objects carry their own namespace attributes:

```python
func1.__name__       # 'func1'
func1.__doc__        # None (no docstring)
func1.__module__     # '__main__'
func1.__code__       # code object (bytecode + metadata)
func1.__defaults__   # tuple of default argument values, or None
func1.__closure__    # tuple of cells for closed-over variables, or None
func1.__annotations__ # dict of parameter/return annotations
func1.__dict__       # custom attributes attached to the function object
```

**`__name__` vs the binding in the namespace:**

```python
func1.__name__ = "func2"   # renames the function object's internal name
func1.__name__    # 'func2'
func2             # NameError — the namespace still has the name 'func1'
func1             # works — the namespace binding is unchanged
```

**Closures:** A function that references variables from an enclosing scope captures them as "cells" in `__closure__`:

```python
def make_adder(n):
    def adder(x):
        return x + n    # 'n' is a free variable
    return adder

add5 = make_adder(5)
add5.__closure__          # (<cell at 0x...>,)
add5.__closure__[0].cell_contents  # 5
add5(10)                  # 15
```

The free variables referenced by a function are listed in `func.__code__.co_freevars`.

---

## 3. Memory Management and Garbage Collection

### 3.1. Reference Counting in CPython

CPython's primary memory management strategy is **reference counting**. Every `PyObject` has an `ob_refcnt` field that tracks how many references exist to it. When a reference is created (assignment, function argument, list append), `ob_refcnt` is incremented via the `Py_INCREF` macro. When a reference is dropped (`del`, function return, reassignment), `ob_refcnt` is decremented via `Py_DECREF`. When `ob_refcnt` reaches zero, `PyObject_Dealloc` is called immediately — the object is freed on the spot, not queued.

```python
import sys

a = []
sys.getrefcount(a)   # 2 (a + the getrefcount argument)

b = a
sys.getrefcount(a)   # 3

del b
sys.getrefcount(a)   # 2

def f(x): pass
f(a)
# inside f: refcount was 3 during the call, drops to 2 after return
```

**Advantages of reference counting:**

- Deterministic finalization — `__del__` is called and memory is freed immediately when the last reference drops.
- No stop-the-world GC pauses for most objects.

**Limitation:** Reference counting cannot collect **reference cycles**:

```python
a = []
a.append(a)   # a contains a reference to itself
del a         # ob_refcnt drops to 1, not 0 — never freed by refcounting alone
```

---

### 3.2. Cycle Detection

CPython includes a supplemental **cycle garbage collector** (in `gc` module) that handles reference cycles. It uses a **tri-color marking** variant.

The cycle collector only tracks **container objects** that can hold references to other objects: `list`, `dict`, `set`, `tuple`, custom class instances, etc. Immutable objects that cannot contain references (small integers, strings) are not tracked.

Each tracked container has a `_gc_prev` / `_gc_next` doubly-linked-list node embedded in a `PyGC_Head` header prepended to the object. The GC periodically walks these lists looking for cycles.

```python
import gc

# Enable/disable the cycle collector
gc.disable()
gc.enable()
gc.isenabled()   # True by default

# Run a collection manually
gc.collect()     # returns number of unreachable objects found

# Inspect tracked objects
gc.get_objects()  # list of all tracked container objects
```

---

### 3.3. Generational Garbage Collection

The cycle collector uses a **three-generation** scheme based on the generational hypothesis (most objects die young):

| Generation | Index | Threshold (new allocations before GC triggered) |
| --- | --- | --- |
| Generation 0 | 0 | 700 (default) — young, recently allocated objects |
| Generation 1 | 1 | 10 — survivors of gen 0 collection |
| Generation 2 | 2 | 10 — long-lived objects (survive gen 1) |

```python
gc.get_threshold()     # (700, 10, 10) by default
gc.set_threshold(1000, 15, 15)   # tune for allocation-heavy workloads

gc.get_count()         # (gen0_count, gen1_count, gen2_count) — current allocation counts
gc.collect(0)          # collect only generation 0
gc.collect(1)          # collect gen 0 + gen 1
gc.collect(2)          # full collection (all generations)
gc.collect()           # same as collect(2)
```

**Lifecycle:** A new object starts in generation 0. If it survives a generation 0 collection, it is promoted to generation 1. If it survives a generation 1 collection, it is promoted to generation 2. Generation 2 objects (long-lived singletons, module objects, class objects) are collected infrequently.

**`__del__` and cycles:** If any object in a reference cycle defines `__del__`, CPython historically could not determine a safe finalization order and moved such cycles to `gc.garbage` (an uncollectable list). As of Python 3.4 (PEP 442), `__del__` objects in cycles are handled correctly — `__del__` is called in an unspecified order, then the cycle is broken.

---

### 3.4. Weak References

A **weak reference** holds a reference to an object without incrementing `ob_refcnt`. The object can be garbage-collected even while weak references to it exist. When the object is collected, the weak reference becomes `None` (or a callback fires).

```python
import weakref

class Expensive:
    def __init__(self, name):
        self.name = name

obj = Expensive("cache-entry")
ref = weakref.ref(obj)

ref()          # <__main__.Expensive object at 0x...>
ref() is obj   # True

del obj
ref()          # None — object was collected
```

**`weakref.WeakValueDictionary`** — a dict where values are weak references:

```python
cache = weakref.WeakValueDictionary()
obj = Expensive("item")
cache["key"] = obj

del obj          # obj is collected
cache.get("key") # None — cleaned up automatically
```

Use cases:

- Caches that should not prevent GC of their entries.
- Observers/callbacks that should not keep the observed object alive.
- Avoiding reference cycles in parent↔child relationships (child holds a weak ref to parent).

---

## 4. Bytecode and the CPython VM

### 4.1. Compilation Pipeline

When Python runs a `.py` file or executes a function, it goes through these stages:

```text
Source (.py)
    ↓ lexer (tokenize)
Tokens
    ↓ parser
AST (Abstract Syntax Tree)
    ↓ compiler (ast → bytecode)
Code Object (bytecode + constants + names)
    ↓ CPython VM (ceval.c)
Execution
```

The AST is visible via the `ast` module:

```python
import ast
tree = ast.parse("x = 1 + 2")
print(ast.dump(tree, indent=2))
```

Compiled bytecode is cached in `__pycache__/<module>.cpython-3XX.pyc` to avoid recompilation on the next import.

---

### 4.2. Inspecting Bytecode with `dis`

The `dis` module disassembles code objects into human-readable bytecode instructions:

```python
import dis

def add(a, b):
    return a + b

dis.dis(add)
# Output:
#   2     RESUME          0
#   3     LOAD_FAST       0 (a)
#         LOAD_FAST       1 (b)
#         BINARY_OP       0 (+)
#         RETURN_VALUE
```

Key opcodes:

| Opcode | What it does |
| --- | --- |
| `LOAD_FAST` | Push a local variable onto the stack |
| `LOAD_GLOBAL` | Push a global/builtin name onto the stack |
| `LOAD_CONST` | Push a constant (integer literal, string literal, etc.) |
| `STORE_FAST` | Pop top of stack, store as a local variable |
| `BINARY_OP` | Pop two items, apply operator, push result |
| `CALL` | Call a callable with N arguments from the stack |
| `RETURN_VALUE` | Return the top of the stack to the caller |
| `BUILD_LIST` | Build a list from N items on the stack |
| `GET_ITER` | Push an iterator for the top-of-stack object |
| `FOR_ITER` | Advance iterator; jump if exhausted |
| `JUMP_BACKWARD` | Loop back to a target instruction |

The CPython VM is a **stack machine** — most instructions operate on a value stack, pushing and popping `PyObject*` pointers.

---

### 4.3. Code Objects

Every function and module has a **code object** (`types.CodeType`) that bundles the bytecode with its metadata:

```python
def greet(name, greeting="Hello"):
    msg = f"{greeting}, {name}!"
    return msg

code = greet.__code__
code.co_name          # 'greet'
code.co_varnames      # ('name', 'greeting', 'msg') — local variable names
code.co_freevars      # () — closed-over variables (non-empty for closures)
code.co_consts        # (None, '{}...')  — constants
code.co_argcount      # 2  — number of positional arguments
code.co_stacksize     # max stack depth needed at runtime
code.co_filename      # source file
code.co_firstlineno   # line number of the def statement
```

Code objects are **immutable** and **shared** — if you define the same lambda in a loop, each call creates a new function object (with potentially different closures) but they all share the same code object.

---

### 4.4. The Global Interpreter Lock (GIL)

CPython has a **Global Interpreter Lock** — a mutex that ensures only one thread executes Python bytecode at a time. It protects CPython's non-thread-safe internals (especially reference counting) from data races.

**Implications:**

| Workload | GIL impact |
| --- | --- |
| **CPU-bound** (computation) | GIL is a bottleneck — threads cannot run Python code in parallel on multiple cores |
| **I/O-bound** (network, file, subprocess) | GIL is released during I/O waits — threads work well |
| **C extensions** | Many (NumPy, Pillow, etc.) release the GIL during pure C computation — parallelism is possible |

**Workarounds for CPU-bound work:**

```python
# multiprocessing — separate processes, each with its own GIL
from multiprocessing import Pool
with Pool(4) as p:
    results = p.map(expensive_function, items)

# concurrent.futures.ProcessPoolExecutor (higher level)
from concurrent.futures import ProcessPoolExecutor
with ProcessPoolExecutor() as ex:
    results = list(ex.map(expensive_function, items))
```

**Python 3.13+ (PEP 703):** The free-threaded build (`--disable-gil`) is experimental. It replaces per-object refcounting with atomic operations and removes the GIL, allowing true multi-core Python execution. Not yet the default.

The GIL is released every `sys.getswitchinterval()` seconds (default: 5ms) or when a blocking I/O call is made, allowing other threads to run.

---

## 5. The Descriptor Protocol

### 5.1. How Attribute Lookup Works

When you access `obj.attr`, Python does not simply look in `obj.__dict__`. The full lookup chain is:

1. Look in `type(obj).__mro__` for a **data descriptor** — an object with both `__get__` and `__set__` (or `__delete__`). If found, call `descriptor.__get__(obj, type(obj))`.
2. Look in `obj.__dict__` (the instance dict). If found, return the value.
3. Look in `type(obj).__mro__` for a **non-data descriptor** or a plain class attribute. If a non-data descriptor, call `__get__`.
4. Raise `AttributeError`.

This order ensures that data descriptors (e.g. `property`) take priority over instance `__dict__` entries, which take priority over non-data descriptors (e.g. functions/methods).

---

### 5.2. Data vs Non-data Descriptors

A **descriptor** is any object that defines `__get__`, `__set__`, or `__delete__` in its class.

| Type | Methods defined | Priority vs instance dict |
| --- | --- | --- |
| **Data descriptor** | `__get__` + `__set__` (and/or `__delete__`) | Higher — overrides instance dict |
| **Non-data descriptor** | Only `__get__` | Lower — instance dict wins |

```python
class Descriptor:
    def __get__(self, obj, objtype=None):
        print(f"__get__ called on {obj}")
        return 42

    def __set__(self, obj, value):
        print(f"__set__ called with {value}")

class MyClass:
    attr = Descriptor()   # class-level descriptor

m = MyClass()
m.attr          # calls Descriptor.__get__(m, MyClass) → 42
m.attr = 99     # calls Descriptor.__set__(m, 99)
```

Functions are **non-data descriptors** — they implement `__get__` to return a bound method object when accessed via an instance. That is how `obj.method()` becomes `MyClass.method(obj)`:

```python
class Foo:
    def bar(self): return self

f = Foo()
Foo.__dict__['bar']        # <function Foo.bar at 0x...>
Foo.__dict__['bar'].__get__(f, Foo)  # <bound method Foo.bar of <Foo ...>>
f.bar                      # same — attribute lookup triggers __get__
```

---

### 5.3. `property` as a Descriptor

`property` is a built-in data descriptor that provides managed attribute access with getter/setter/deleter semantics:

```python
class Circle:
    def __init__(self, radius):
        self._radius = radius

    @property
    def radius(self):
        return self._radius

    @radius.setter
    def radius(self, value):
        if value < 0:
            raise ValueError("radius must be non-negative")
        self._radius = value

    @property
    def area(self):
        import math
        return math.pi * self._radius ** 2

c = Circle(5)
c.radius        # 5 — calls getter
c.radius = 10   # calls setter
c.radius = -1   # ValueError
c.area          # 314.159...
```

Under the hood, `@property` creates a `property` object with `__get__`, `__set__`, and `__delete__` methods. Because `property` is a data descriptor, `c.radius` calls `property.__get__` even if `c.__dict__` has a key `'radius'`.

---

### 5.4. `__slots__`

By default, each instance stores its attributes in a per-instance `__dict__`. This dict has overhead: ~200–300 bytes for an empty dict, plus pointer storage per key.

`__slots__` replaces the per-instance `__dict__` with a fixed C array of slots — one per declared attribute:

```python
class Point:
    __slots__ = ('x', 'y')

    def __init__(self, x, y):
        self.x = x
        self.y = y

p = Point(1, 2)
p.x         # 1
p.z = 3     # AttributeError — z is not in __slots__
p.__dict__  # AttributeError — no __dict__ on slots-only class
```

Each slot is implemented as a **data descriptor** at the class level — that is why attribute access is fast and why undeclared attributes are rejected.

**Memory comparison:**

```python
import sys

class WithDict:
    def __init__(self): self.x = self.y = 0

class WithSlots:
    __slots__ = ('x', 'y')
    def __init__(self): self.x = self.y = 0

sys.getsizeof(WithDict())    # ~48 + dict overhead (~200)
sys.getsizeof(WithSlots())   # ~56 — compact, no __dict__
```

**Tradeoffs:**

| Concern | `__dict__` | `__slots__` |
| --- | --- | --- |
| Memory per instance | Higher (dict overhead) | Lower (C array) |
| Attribute access speed | Slightly slower (dict lookup) | Faster (offset into array) |
| Dynamic attributes | Allowed | Not allowed |
| Multiple inheritance | Works naturally | Careful: all classes in MRO must cooperate |
| Pickling | Works by default | Requires `__getstate__`/`__setstate__` |

Use `__slots__` when creating millions of instances and memory is a concern (e.g. data records, AST nodes, game entities).

---

## 6. Metaclasses

### 6.1. `type` is the Metaclass of All Classes

A **metaclass** is the class of a class. Just as an object is an instance of its class, a class is an instance of its metaclass. The default metaclass for all classes is `type`.

```python
class Foo: pass

type(Foo)        # <class 'type'>
type(type)       # <class 'type'>   — type is its own metaclass
type(object)     # <class 'type'>
isinstance(Foo, type)   # True

# Creating a class with type() directly
Bar = type('Bar', (object,), {'x': 42, 'greet': lambda self: 'hi'})
Bar.x        # 42
Bar().greet() # 'hi'
```

`type(name, bases, namespace)` — this three-argument form dynamically creates a new class. The `class` statement is syntactic sugar for calling the metaclass.

---

### 6.2. `__new__` vs `__init__`

`__new__` allocates and returns the new instance; `__init__` initializes it. `__new__` runs first.

```python
class MyClass:
    def __new__(cls, *args, **kwargs):
        print(f"__new__ called, cls={cls}")
        instance = super().__new__(cls)
        return instance

    def __init__(self, value):
        print(f"__init__ called, value={value}")
        self.value = value

obj = MyClass(42)
# __new__ called, cls=<class 'MyClass'>
# __init__ called, value=42
```

Key distinction:

| | `__new__` | `__init__` |
| --- | --- | --- |
| **Purpose** | Allocate and return the instance | Configure the already-allocated instance |
| **Receives** | `cls` (the class) | `self` (the new instance) |
| **Must return** | The new instance (or None to skip `__init__`) | Nothing (`None`) |

`__new__` is used for:

- **Singletons** — return the existing instance if one already exists.
- **Immutable types** — `int`, `str`, `tuple` are immutable; customization must happen in `__new__` since by the time `__init__` runs, the value is already set.
- **Custom memory pools or instance caches.**

```python
# Singleton via __new__
class Singleton:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

a = Singleton()
b = Singleton()
a is b   # True
```

---

### 6.3. Custom Metaclasses

A custom metaclass can intercept class creation — useful for registering subclasses, enforcing interfaces, adding methods, etc.

```python
class RegistryMeta(type):
    registry = {}

    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
        if bases:   # skip the base class itself
            RegistryMeta.registry[name] = cls
        return cls

class Plugin(metaclass=RegistryMeta):
    pass

class AudioPlugin(Plugin):
    pass

class VideoPlugin(Plugin):
    pass

RegistryMeta.registry
# {'AudioPlugin': <class '__main__.AudioPlugin'>, 'VideoPlugin': <class '__main__.VideoPlugin'>}
```

Metaclass hooks:

| Hook | When it runs | Use for |
| --- | --- | --- |
| `__prepare__(mcs, name, bases)` | Before the class body executes | Return a custom namespace dict (e.g. `OrderedDict`) |
| `__new__(mcs, name, bases, ns)` | After class body executes | Create and return the class object |
| `__init__(cls, name, bases, ns)` | After `__new__` | Additional initialization |
| `__call__(cls, *args, **kwargs)` | When the class is called to create instances | Override instance creation |

Modern alternative: for many metaclass use cases, `__init_subclass__` (PEP 487, Python 3.6) is simpler:

```python
class Base:
    subclasses = []

    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        Base.subclasses.append(cls)

class A(Base): pass
class B(Base): pass

Base.subclasses   # [<class 'A'>, <class 'B'>]
```

`__init_subclass__` is called on the base class whenever a subclass is defined — no metaclass needed.

---

## Sources

- [Python Data Model](https://docs.python.org/3/reference/datamodel.html) — official reference
- [Python Internals blog](https://realpython.com/cpython-source-code-guide/) — CPython source walkthrough
- [PEP 3135](https://peps.python.org/pep-3135/) — `super()` and `__class__`
- [PEP 442](https://peps.python.org/pep-0442/) — Safe object finalization (`__del__` with cycles)
- [PEP 487](https://peps.python.org/pep-0487/) — `__init_subclass__` and `__set_name__`
- [PEP 703](https://peps.python.org/pep-0703/) — Making the GIL optional
- [Fluent Python](https://www.oreilly.com/library/view/fluent-python-2nd/9781492056348/) — Luciano Ramalho
