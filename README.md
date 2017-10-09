<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Python - Back to basics](#python---back-to-basics)
    - [1. Python and Objects](#1-python-and-objects)
      - [1.1. Everything in Python is an object.](#11-everything-in-python-is-an-object)
      - [1.2. Objects and Attributes](#12-objects-and-attributes)
      - [1.3. Types with `type()`](#13-types-with-type)
      - [1.4. Method Resolution Order [MRO]](#14-method-resolution-order-mro)
      - [1.5. Inheritance, and Method Resolution Order](#15-inheritance-and-method-resolution-order)
      - [1.6. Callables](#16-callables)
      - [1.7. Object size](#17-object-size)
    - [2. Names and Namespaces](#2-names-and-namespaces)
      - [2.1. Names and Namespaces](#21-names-and-namespaces)
      - [2.2. `id()`, `is` and `==`](#22-id-is-and-)
      - [2.3. Object Attributes and NameSpaces](#23-object-attributes-and-namespaces)
        - [2.3.1. Creating a custom namespace](#231-creating-a-custom-namespace)
      - [2.4. Object Reference count](#24-object-reference-count)
      - [2.5. Various methods to set names in a namespace](#25-various-methods-to-set-names-in-a-namespace)
        - [2.5.1. Direct assignment](#251-direct-assignment)
        - [2.5.2. Tuple unpacking](#252-tuple-unpacking)
        - [2.5.3. Extended iterable tuple unpacking (only in Python3)](#253-extended-iterable-tuple-unpacking-only-in-python3)
      - [2.6. Overwriting builtin names](#26-overwriting-builtin-names)
      - [2.7. Function locals](#27-function-locals)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Python - Back to basics

***

### 1. Python and Objects

#### 1.1. Everything in Python is an object.

Anything that is created by Python, is an instance of a inbuilt type.

The newly created variable is a reference in the current namespace, to an object (a blob with some metadata) in memory.

Hence if a new variable is created, for example `v = 1`, the following happens:

1. The Python interpreter finds the appropriate in-built data type that can represent the data input. ie.. int(), float(), a function, class() etc..
2. An instance of the appropriate type class is spawned in memory, which has a specific ID, and is assigned the value.
3. The instance inherits the attributes of the type class.
4. A pointer is created in the current namespace with the name `v`, that points to the instance in memory.

Thus, when creating a variable `v = 1`, `v` is a reference to the object in memory created by inheriting from the builtin `int` type.

Every object has:

1. A single type (ie.. every object is an instance of an inbuilt type (class) like int, float etc.. (which is a class)
2. A single value
3. Attributes, mostly inherited from the builtin type
4. One or more base classes (The object is an instance of a builtin class, hence it inherits from it as well)
5. A single unique ID (Since an object is an instance of a class, it is a running copy in memory and has an id)
6. One or more names, in one or more namespaces (The object created in memory has a reference to it in the namespace)

***

#### 1.2. Objects and Attributes

Object attributes are inherited from the class from which it was instantiated, through classes in the MRO chain, as well as its parent classes.

To list the methods available for an object, use the `dir()` function on the object.

```python3
In [36]: dir(a)
Out[36]:
['__abs__',
 '__add__',
 '__and__',
 '__bool__',
 '__ceil__',
..
....
<omitted>
```

***

#### 1.3. Types with `type()`

For example,

```python3
In [14]: type(1)
Out[14]: int

In [15]: type(int)
Out[15]: type

In [16]: help(int)
Help on class int in module builtins:

class int(object)
 |  int(x=0) -> integer
 |  int(x, base=10) -> integer
 |
 |  Convert a number or string to an integer, or return 0 if no arguments
 |  are given.  If x is a number, return x.__int__().  For floating point
 |  numbers, this truncates towards zero.
 |
 |  If x is not a number or if base is given, then x must be a string,
 |  bytes, or bytearray instance representing an integer literal in the
 |  given base.  The literal can be preceded by '+' or '-' and be surrounded
 |  by whitespace.  The base defaults to 10.  Valid bases are 0 and 2-36.
 |  Base 0 means to interpret the base from the string as an integer literal.
 |  >>> int('0b100', base=0)
 |  4
```

When the python interpreter calls the `type()` function on a variable or a builtin, it does the following:

1. The `type()` function follows the variable name or builtin name to the actual object in memory.
2. It reads the object metadata and calls the magic method `__class__` on it.
3. This prints the type of the class from which the object was created, which is of course, the class of the object as well.

> In the example above, the integer `1` is an instance of the inbuilt type `int`.

**IMPORTANT**

1. Every object that is created by Python is an instance of an inbuilt type.
2. Every type inherits from another type which ultimately ends by inheriting from the `object` type.

**NOTE:**

* Calling type(`something`) internally calls `something.__class__`.

```python3
In [1]: type(1)
Out[1]: int

In [2]: (1).__class__
Out[2]: int
```

* If you call `type()` on an inbuilt such as `int`,  it returns `type` which means it's a base type.

***

#### 1.4. Method Resolution Order [MRO]

`Method Resolution Order` is the order in which a method is resolved.

When a method is called on an object, it first looks up in the inherited methods (from the class from which the object was instantiated), and if not found, moves to its parent class.

Hence, an integer object will first look for the methods under the `int()` class, and then the parent class of `int()`, ie.. object().

**Code example**:

```python
In [18]: a = 1

In [19]: a
Out[19]: 1

In [20]: a.__mro__
---------------------------------------------------------------------------
AttributeError                            Traceback (most recent call last)
<ipython-input-20-bc8e99ec9963> in <module>()
----> 1 a.__mro__

AttributeError: 'int' object has no attribute '__mro__'

In [21]: int.__mro__
Out[21]: (int, object)
```

**NOTE:**
* In the example above, `a.__mro__` will fail, since the `__mro__` method is not available on objects or its parent class.

The actual way to get the Method Resolution Order, is to use the `inspect` module

```python3
In [33]: import inspect

In [34]: inspect.getmro(int)
Out[34]: (int, object)

In [35]: inspect.getmro(type(a))
Out[35]: (int, object)
```

**Note**

* Every object has one or more base classes.
* Every object created is an instance of a class which is either inbuilt like `int` or a custom made class.
* All classes whether custom or inbuilt, ultimately inherits from the `object` class.

-TODO-: Other than doing `import inspect; inspect.getmro(int)` how does `__mro__` gets executed when called as a magic method?

***

#### 1.5. Inheritance, and Method Resolution Order

The `__class__` method is implemented for almost all the type classes which inherits from the `object` class.

This allows to probe the `type` and other internals such as the MRO.

```python3
In [98]: True.__mro__
---------------------------------------------------------------------------
AttributeError                            Traceback (most recent call last)
<ipython-input-99-89beb515a8b6> in <module>()
----> 1 True.__mro__

In [99]: type(True)
Out[99]: bool

In [100]: True.__class__
Out[100]: bool

In [101]: True.__class__.__bases__
Out[101]: (int,)

In [102]: True.__class__.__bases__[0]
Out[102]: int

In [103]: True.__class__.__bases__[0].__bases__
Out[103]: (object,)
```

To understand the inheritance, we try checking the type or the inbuilt `True` condition. We find that `True.__mro__` does not exist. This is because it's an instance of another class.

To find it, we can use either `type()` or `True.__class__`. This will print the class that it inherits from. Here, it's the class `bool`.

If we use `True.__class__.__bases__`, the python interpreter will show the base class of the class the instance is inheriting from, which is `int` here. Hence `True` is an instance of `bool`, and `bool` inherits from `int`.

`True.__class__.bases__[0].__bases__` should print the base class of `int`, ie.. the `object` class.

A second example:

```python3
In [128]: j = 2

In [129]: type(j)
Out[129]: int

In [130]: j.__class__
Out[130]: int

In [131]: j.__class__.__base <TAB>
j.__class__.__base__   j.__class__.__bases__

In [131]: j.__class__.__base__
Out[131]: object

In [132]: j.__class__.__bases__
Out[132]: (object,)
```

1. Define a variable `j` with a value `2, which creates an instance of the `int` class.
2. Confirm this using `type()` or `instance.__class__`.
3. Inspect the base class of `j` using `j.__class__.__base__` or `j.__class__.__bases__`

`j.__class__.__base__` will show a single parent class, while `j.__class__.__bases__` shows if there are multiple parent classes.

**NOTE:**
* Hence, `j` is an instance of class `int`, and it inherits from class `object`.
* Probing for the base class of `object` won't print anything since `object` is the ultimate base class.

The same information can be pulled using the `getmro()` method in the `inspect` module.

```python3
In [37]: type(bool)
Out[37]: type

In [38]: inspect.getmro(bool)
Out[38]: (bool, int, object)
```

***

#### 1.6. Callables

_Read more on Callables at [https://arvimal.blog/2017/08/09/callables-in-python/](https://arvimal.blog/2017/08/09/callables-in-python/)_

Instance objects are not callable, only functions, classes, or methods are callable.

This means, the function/method/class or any object can be executed and returns a value (can be `False` as well)

```python3
In [160]: x = int(1212.3)

In [161]: y = "Hello"

In [162]: callable(x)
Out[162]: False

In [163]: callable(y)
Out[163]: False

In [164]: class MyClass(object):
   .....:     pass
   .....:

In [165]: callable(MyClass)
Out[165]: True

In [166]: def myfunc():
   .....:     pass
   .....:

In [167]: callable(myfunc)
Out[167]: True
```

***

#### 1.7. Object size

Object size in memory can be parsed using the `getsizeof` method from the `sys` module

```python3
In [174]: import sys

In [175]: sys.getsizeof("Hello")
Out[175]: 54

In [176]: sys.getsizeof(2**30 + 1)
Out[176]: 32
```

The help on `sys` shows:

```python3
In [42]: help(sys.getsizeof)

Help on built-in function getsizeof in module sys:

getsizeof(...)
    getsizeof(object, default) -> int

    Return the size of object in bytes.
```

***

### 2. Names and Namespaces

#### 2.1. Names and Namespaces

1. A Name is a mapping to a value, ie.. a reference to objects in memory.

2. A Namespace is similar to a dictionary, ie.. it is a set of valid identifier names to object references.

3. Operations such as assignment (=), renaming, and `del` are all namespace operations.

4. A **_scope_** is a section where a namespace is directly accessible, for example, `dir()` shows the current namespace scope.

5. Dot notations ('.') are used to access in-direct namespaces:
    sys.version_info.major
    p.x
    "Hello".__add__(" World!")

6. It's better not to overwrite builtin names with custom ones, unless there is a strong reason to do so.

**NOTE:**
>A Namespace cannot carry more than one similar name.
>As an example, multiple variables named `a` cannot exist in a namespace.

_Read more on Python Namespaces at [Python3 Classes documentation](https://docs.python.org/3/tutorial/classes.html#python-scopes-and-namespaces)_

A `dir()` function can list the names in the current namespace.

* Example 1

```python3
In [46]: dir()
Out[46]:
['In',
 'Out',
 '_',
 '_1',
 '_11',
 '_14',
 '_19',
 '_2',
 '_21',
 '_26',
...
....
 '_iii',
 '_oh',
 '_sh',
 'a',
 'exit',
 'get_ipython',
 'inspect',
 'quit',
 'sys']
```

Some important points on Names:

1. A name assignment (Creating a variable), renaming, deleting etc.. are all namespace operations.
2. Python uses names as a _reference to objects in memory_, and not like boxes containing a value.
3. Variable names are actually labels which you can add (or remove) to an object.
4. Deleting a name just removes the reference in the current namespace to the object in memory.
5. When all references (names) are removed from the namespace that refers a specific object, the object is garbage collected.
6. It's not names (variables) that have types but objects, since objects are actually instances of specific classes (int, str, float etc..)
7. Due to `**6**`, the same name which was referring to an int can be assigned to a str object

* What happens when `a = 10` in a python REPL prompt?

1. The python interpreter creates an object in memory which is an instance of `class int`.
2. It then creates a name called `a` which is a pointer to the object instance.

* Example 1

```python
In [7]: a = 300

In [8]: a
Out[8]: 300

In [9]: a = 400

In [10]: a
Out[10]: 400
```

Explanation:

1. An object of type `int` is created in memory and assigned a value of `300`.
2. A name `a` is created in the current namespace and points to the address of the object.
3. Hence, when `a` is called from the prompt, the interpreter fetches the content from memory, ie.. `300`.

4. When `a` is assigned `400`, a new object is created in memory with a value of `400`.
5. The name `a` in the namespace is now set to point to the new object `400`.
6. Since the object with value `300` is not referenced anymore, it is garbage collected.

***

#### 2.2. `id()`, `is` and `==`

The builtins `id()`, as well as `is` and `==` are valuable to understand the semantics of names in a namespace.

* Example 1

```python
In [26]: a = 400

In [27]: a
Out[27]: 400

In [28]: b = a

In [29]: b
Out[29]: 400

In [30]: id(a)
Out[30]: 139736842153872

In [31]: id(b)
Out[31]: 139736842153872

In [32]: a == b
Out[32]: True

In [33]: a is b
Out[33]: True

In [41]: del b

In [42]: b
---------------------------------------------------------------------------
NameError                                 Traceback (most recent call last)
<ipython-input-42-3b5d5c371295> in <module>()
----> 1 b

NameError: name 'b' is not defined

In [43]: a
Out[43]: 400

In [44]: del a

In [45]: a
---------------------------------------------------------------------------
NameError                                 Traceback (most recent call last)
<ipython-input-45-60b725f10c9c> in <module>()
----> 1 a

NameError: name 'a' is not defined
```

Explanation:

1. Created an object of value `400` and assigned it a name `a`.
2. Created another namespace variable `b` and assigned it to be `a`. This makes `b` refer the same address `a` refers to.
3. Since both `a` and `b` refers to the same address and hence the same object, both `id(a)` and `id(b)` are same.
4. Hence, `a == b` and `a is b` are same as well.
5. `del b` deletes the name from the namespace, and does not touch the object in memory.
6. Since `a` still refers to the object, it can be accessed by calling `a`.
7. When `del a` is executed, it removes the existing reference to the object.
8. Once no more references exist in the namespace to an object in memory, the object is garbage-collected.

>**IMPORTANT:**
> `a == b` evaluates the **value of the objects** that `a` and `b` refers to.
> `a is b` evaluates the **address of the objects** that `a` and `b` refers to.

>ie.. `a == b` check if both `a` and `b` has the same value
> while `a is b` checks if both `a` and `b` refers to the exact same object (same address).

* Can we use the same name in a namespace, for a different object type?

```python
In [51]: a = 10

In [52]: id(a)
Out[52]: 139737075565888

In [53]: a
Out[53]: 10

In [54]: a = "Walking"

In [55]: id(a)
Out[55]: 139736828783896

In [56]: a
Out[56]: 'Walking'
```

Assigning an existing name to another value/type is possible. It sets the pointer to the new type, which is an entirely different object altogether.

When a new object is assigned to an existing name in the namespace, it changes the reference to the new object, and no longer reference the old object.

>**NOTE:**
> A single name in the namespace cannot refer to multiple objects in memory, just like a file name cannot refer to multiple file content.

***

#### 2.3. Object Attributes and NameSpaces

Objects get their attributes from the base type the object is instantiated from.

Object attributes are similar to dictionaries, since the attributes of an object are names to methods within the object namespace.

```python3
In [19]: a = "test"

In [20]: a.__class__
Out[20]: str

In [21]: dir(a)
Out[21]:
['__add__',
 '__class__',
 '__contains__',
 '__delattr__',
 '__dir__',
 '__doc__',
 '__eq__',
 '__format__',
 '__ge__',
 '__getattribute__',
...
.....
 'startswith',
 'strip',
 'swapcase',
 'title',
 'translate',
 'upper',
 'zfill']
```

##### 2.3.1. Creating a custom namespace

Creating a custom name-space can help in understanding the concept better.

In Python v3.3, the `types` module include a class named `SimpleNamespace` which provides a clean namespace to play with.

```python
from types import SimpleNamespace
```

* In Python v2, it's equivalent to the following code, ie.. creating a Class and setting the attributes

```python
class SimpleNamespace(object):
    pass
```

* New methods can be assigned in this namespace without overriding any existing ones.

```python
In [64]: from types import SimpleNamespace

In [65]: ns = SimpleNamespace()

In [66]: ns
Out[66]: namespace()

In [67]: ns.a = "A"

In [68]: ns.b = "B"

In [69]: ns
Out[69]: namespace(a='A', b='B')

In [70]: ns.
ns.a  ns.b

In [70]: ns.a
Out[70]: 'A'

In [71]: ns.b
Out[71]: 'B'

In [72]: ns.__dict__
Out[72]: {'a': 'A', 'b': 'B'}
```

The `__dict__` special method is available in both Python v2 and v3, for the object, and it returns a dictionary.

***

#### 2.4. Object Reference count

* The class `getrefcount` from the module `sys` helps in understanding the references an object has currently.

```python
In [2]: from sys import getrefcount

In [3]: getrefcount(None)
Out[3]: 12405

In [4]: getrefcount(int)
Out[4]: 113

In [5]: a = 10

In [6]: getrefcount(a)
Out[6]: 113

In [7]: b = a

In [8]: getrefcount(a)
Out[8]: 114

In [9]: getrefcount(b)
Out[9]: 114
```

* Python pre-creates many integer objects for effeciency reasons (Mostly speed)

```python
In [1]: from sys import getrefcount

In [2]: [(i, getrefcount(i)) for i in range(20)]
Out[2]:
[(0, 2150),
 (1, 2097),
 (2, 740),
 (3, 365),
 (4, 366),
 (5, 196),
 (6, 173),
 (7, 119),
 (8, 248),
 (9, 126),
 (10, 114),
 (11, 110),
 (12, 82),
 (13, 55),
 (14, 50),
 (15, 62),
 (16, 150),
 (17, 52),
 (18, 42),
 (19, 45)]
```

#### 2.5. Various methods to set names in a namespace

There are multiple ways to set a name for an object, in a namespace.

##### 2.5.1. Direct assignment

```python3
In [42]: a = 1

In [43]: b = a

In [44]: c = b = a

In [45]: c
Out[45]: 1

In [46]: b
Out[46]: 1

In [47]: a
Out[47]: 1
```

##### 2.5.2. Tuple unpacking

Multiple names can be assigned in a single go, if the RHS values correspond the LHS names.

RHS has to be iterable so that the assignment will work properly. The RHS can be a list, a tuple, or a series of data.

* Calling the names one-by-one unpacks the tuple and returns the single value.
* When all the names are called simultaneously, they are returned as a tuple.


```python3
In [48]: a, b, c = 10, 20, 30

In [49]: a
Out[49]: 10

In [50]: b
Out[50]: 20

In [51]: c
Out[51]: 30

In [52]: a, b, c
Out[52]: (10, 20, 30)
```

##### 2.5.3. Extended iterable tuple unpacking (only in Python3)

This feature exists only in Python v3.

The examples below are self-explanatory.

* Example 1

```python3
In [54]: a, b, c, *d = "HelloWorld!"

In [55]: a
Out[55]: 'H'

In [56]: b
Out[56]: 'e'

In [57]: c
Out[57]: 'l'

In [58]: d
Out[58]: ['l', 'o', 'W', 'o', 'r', 'l', 'd', '!']
```

* Example 2:

```python3
In [59]: a, *b, c = "HelloWorld"

In [60]: a
Out[60]: 'H'

In [61]: b
Out[61]: ['e', 'l', 'l', 'o', 'W', 'o', 'r', 'l']

In [62]: c
Out[62]: 'd'

In [64]: a, b, c
Out[64]: ('H', ['e', 'l', 'l', 'o', 'W', 'o', 'r', 'l'], 'd')
```

* Example 3:

```python3
In [65]: a, *b, c = "Hi"

In [66]: a
Out[66]: 'H'

In [67]: c
Out[67]: 'i'

In [68]: b
Out[68]: []

In [69]: a, b, c
Out[69]: ('H', [], 'i')
```

***

#### 2.6. Overwriting builtin names

It is not a good practice to overwrite builtin names, since programs may start acting weirdly.

For example, `len()` calls the dunder method `__len__()` on the object (provided the type supports `len()`), and returns the length. Imagine overwriting it with a custom value.

```python3
In [4]: a = "A"

In [5]: len(a)
Out[5]: 1

In [6]: len = "Hello"

In [7]: len(a)
---------------------------------------------------------------------------
TypeError                                 Traceback (most recent call last)
<ipython-input-7-af8c77e09569> in <module>()
----> 1 len(a)

TypeError: 'str' object is not callable
```

Fortunately, overwriting `len` only means that it hides the builtin with the custom assignment. Deleting the custom assignment will re-instate it to the previous state, by unhiding it. The name will resolve to the builtins since it can't find the name in the local namespace.

Continuing from the previous assignments:

```python3
In [8]: del len

In [9]: len(a)
Out[9]: 1
```

***

#### 2.7. Function locals

