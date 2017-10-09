<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Python - Back to basics](#python---back-to-basics)
  - [Table of Contents](#table-of-contents)
    - [1. Python and Objects](#1-python-and-objects)
      - [1. Everything in Python are objects, how?](#1-everything-in-python-are-objects-how)
      - [2. Every object has:](#2-every-object-has)
      - [How does creating a variable `v = 1` work?](#how-does-creating-a-variable-v--1-work)
      - [What does it mean when the Python interpreter prints the type of a variable (or other objects)?](#what-does-it-mean-when-the-python-interpreter-prints-the-type-of-a-variable-or-other-objects)

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

#### 1.2. Every object has:

1. A single type (ie.. every object is an instance of an inbuilt type (class) like int, float etc.. (which is a class)
2. A single value
3. Attributes, mostly inherited from the builtin type
4. One or more base classes (The object is an instance of a builtin class, hence it inherits from it as well)
5. A single unique ID (Since an object is an instance of a class, it is a running copy in memory and has an id)
6. One or more names, in one or more namespaces (The object created in memory has a reference to it in the namespace)


#### 2.2. How does creating assigning a variable work?
1. When it sees the literal `1`, the python interpreter checks for the best fit builtin object type.
2. In this case, the interpreter arrives at a conclusion that it's an `int`.
3. The interpreter creates the object in memory by inheriting from the builtin `int` type.
4. It creates a reference named `v` to the newly created `int` object, in the current namespace.


#### 2.3. What does it mean when the Python interpreter prints the type of a variable (or other objects)?

For example,

```python3
In [14]: type(1)
Out[14]: int

In [15]: type(int)
Out[15]: type
```

**Ans:** When the python interpreter runs the `type()` function on a variable or a builtin, it's doing the following:

1. The type function follows the variable name or builtin name to the actual object.
2. It reads the object metadata and prints the type.

In the above example, this is what `type(1)` does.

But if you run `type()` on an inbuilt function such as `int`,  it returns `type` which means it's a base type.


> In the example above, the integer `1` is an instance of the inbuilt type `int`.

**IMPORTANT**

1. Every object that is created by Python is an instance of an inbuilt type.
2. Every type inherits from another type which ultimately ends by inheriting from the `object` type.


**Code example**:
```python
In [25]: v = 1

In [26]: type(v)
Out[26]: int

In [27]: v.__mro # (Does not tab complete)

In [28]: int.__mro__
Out[28]: (int, object)
```

The above example shows the following:

* We create an object of type `int` and assigns it a reference names `v` in the current namespace.
* This new object with name `v` inherits from the `int` builtin type, shown by the `type()` output.
* This new object named `v` is hence an instance of the builtin type `int`.
* The `__mro__` method is a way to understand the object inheritance order.
* The instance `v` doesn't have `__mro__` since it's an instance created from a class `int`.
* We can confirm that `int` is indeed a class by using `help(int)` which opens up the help showing it's a class.
* _Instances don't have a `__mro__` magic method, but classes do_.
* Hence, an `int.__mro__` shows how the `int` class inherits from the `object` class

**Note**

* Every object has one or more base classes.
* Every object created is an instance of a class which is either inbuilt like `int` or a custom made class.
* All classes whether custom or inbuilt, ultimately inherits from the `object` class.

Check the example below to understand more:

```python
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

To understand the inheritance, we try checking the type or the inbuilt `True` condition.

We find that `True.__mro__` does not exist. This is because it's an instance of another class.

To find it, we can use either `type()` or `True.__class__`. This will print the class that it inherits from. Here, it's the class `bool`.

If we use `True.__class__.__bases__`, the python interpreter will show the base class of the class the instance is inheriting from, which is `int` here. Hence `True` is an instance of `bool`, and `bool` inherits from `int`.

`True.__class__.bases__[0].__bases__` should print the base class of `int`, ie.. the `object` class.

***

**Another example:**

```python
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

In [133]: j.__class__.__base__.__bases__
Out[133]: ()

In [134]: j.__class__.__base__.__base__

In [135]:
```

* We define a variable `j` with a value `2.
* It creates an instance of `int` class, which we can see using `type(j)` or `j.__class__`.
* To see the base class of `j`, we can use `j.__class__.__base__` or `j.__class__.__bases__`
	* The first one will show just a single parent class, while the second can show if there are multiple base classes.
	* We end up understanding that `j` is an instance of class `int`, which inherits from class `object`.
* If we again go ahead with finding the base class of `object`, we find it's empty since `object` is the most base.


***

> **NOTE**: The module `inspect` is a very helpful one in understanding various features of an object.

**Another example**

```python
In [146]: j = 2

In [149]: inspect.getmro(j)
---------------------------------------------------------------------------
AttributeError                            Traceback (most recent call last)
<ipython-input-149-24257c6e2e0d> in <module>()
----> 1 inspect.getmro(j)

/usr/lib64/python3.5/inspect.py in getmro(cls)
    441 def getmro(cls):
    442     "Return tuple of base classes (including cls) in method resolution order."
--> 443     return cls.__mro__
    444
    445 # -------------------------------------------------------- function helpers

AttributeError: 'int' object has no attribute '__mro__'

In [150]: inspect.getmro(type(j))
Out[150]: (int, object)
```

* The method `inspect.getmro(<x>)` prints the inheritance order of an object. Since `j` is an instance here, it doesn't have an mro.

* But if we use `type(j)`, it actually calls the type (ie.. `int` in our case) and runs on `inspect.getmro()` on it.

* Since `int` is a class [See `help(int)`], `inspect.getmro()` returns the MRO properly.

***

* Callable

Instance objects are not callable, only classes or functions are since it can be called and it returns something.

```python
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

* How to get the memory size of an object?

```python
In [174]: import sys

In [175]: sys.getsizeof("Hello")
Out[175]: 54
```

The above example doesn't just show the size of integer, but it actually shows the size of the instance object created from the `int` class.

```python
In [177]: sys.getsizeof(2**30 + 1)
Out[177]: 32
```

***

### 2. Names and Namespaces

1. A name assignment (Creating a variable), renaming, deleting etc.. are all namespace operations.

2. Python uses names as a reference to objects in memory, and not like boxes containing a value.

3. Variable names are actually labels which you can add (or remove) to an object.

4. Deleting a name just removes the refernce in the current namespace to the object in memory.

5. When all references (names) are removed from the namespace that refers a specific object, the object is garbage collected.

6. It's not names (variables) that have types but objects, since objects are actually instances of specific classes (int, str, float etc..)

7. Due to `**6**`, the same name which was referring to an int can be assigned to a str object
**Q.** What happens when you do `a = 10` in a python REPL prompt?

* The python interpreter creates an object in memory which is an instance of `class int`.
* It then creates a name called `a` which is a pointer to the object instance.

**Q.** What happens with the following assignments?

```python
In [7]: a = 300

In [8]: a
Out[8]: 300

In [9]: a = 400

In [10]: a
Out[10]: 400
```

The following things happen with the above code:

* An object of type `int` is created in memory and assigned a value of `300`.
* A name `a` is created in the current namespace and points to the address of the object.
* Hence, when `a` is called from the prompt, the interpreter fetches the content from memory.

* When `a` is assigned `400`, a new object is created in memory with a value of `400`.
* The name `a` is removed, and is newly created to point to the new object instance of `400`.
* Since the object with value `300` is not referenced anymore, it is garbage collected.

**Q.** Explain what happens with the following code:

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

* We create an object of value `400` and give it a name `a`.
* We create another namespace variable and assign it `a`.
* This make `b` refer to the same address  that `a` refers.
* Since both `a` and `b` refers to the same address and hence the same object, both `id(a)` and `id(b)` are same.
* Hence, `a == b` and `a is b` are same as well.

>**NOTE:**
> `a == b` evaluates the **value of the objects** that `a` and `b` refers to.
> `a is b` evaluates the **address of the objects** that `a` and `b` refers to.

>ie.. `a == b` check if both `a` and `b` has the same value
> while `a is b` checks if both `a` and `b` refers to the exact same object (same address).

* `del b` deletes the name `b` from the namespace.
* Since `a` still refers to the object, it can still be accessed through `a`.
* When `del a` is run, it removes the existing reference to the object, and thus there exists no more references.
* This will garbage collect the object in memory.

* Can we use the same name in a namespace for a different object type?

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

It is absolutely fine to assign the same name in a namespace to a different object, as in the example above.

When you assign a new object to an existing name in the namespace, it just changes its reference to the new object. It no longer reference the old object.

>**NOTE:**
> A single name in the namespace cannot refer to multiple objects in memory, just like a file name cannot refer to multiple file content.


***

#### 2.2. The `SimpleNamespace` class

The module `types` in Python v3 comes with a class named `SimpleNamespace` which gives us a clean namespace to play with.

```python
from types import SimpleNamespace
```

* In Python v2, it's equivalent to the following code:

```python
class SimpleNamespace(object):
    pass
```

* New methods can be assigned in this namespace without the fear of overriding any existing ones.

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

In [72]: ns.__
ns.__class__         ns.__getattribute__  ns.__reduce__
ns.__delattr__       ns.__gt__            ns.__reduce_ex__
ns.__dict__          ns.__hash__          ns.__repr__
ns.__dir__           ns.__init__          ns.__setattr__
ns.__doc__           ns.__le__            ns.__sizeof__
ns.__eq__            ns.__lt__            ns.__str__
ns.__format__        ns.__ne__            ns.__subclasshook__
ns.__ge__            ns.__new__
```

***

#### 2.2. Object Reference count

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

* Python pre-creates many integer objects for effeciency reasons (Still not sure what effeciency)

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

***

#### 2.3. NameSpaces

1. A namespace is a mapping of valid identifier names to objects. The objects may either exist in memory, or will be created at the time of assignment.

2. Simple assignment (=), renaming, and `del` are all namespace operations.

3. A **scope** is a section on Python code where a namespace is directly accessible.

4. Dot notations ('.') are used to access in-direct namespaces:
	* sys.version_info.major
	* p.x

5.



***

#### 2.4. Importing modules into namespaces


***

**Functions**
