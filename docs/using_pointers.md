# Using Pointers

## Creation

To create a pointer, you can use `to_ptr`, like so:

```py
from pointers import to_ptr

ptr = to_ptr("hello world")
```

You can also use the `_` object to replicate the address-of operator, like in other languages:

```py
from pointers import _

ptr = _&"hello world"
```

Finally, you can directly call `Pointer.make_from`:

```py
from pointers import Pointer

ptr = Pointer.make_from("hello world")
```

**Note:** `Pointer.make_from` is more a low level function for creating a pointer. Its API may be changed at any time without warning.

## Dereferencing

There are a few ways to get the underlying value of the pointer, the simplest being calling the `dereference()` method, like so:

```py
from pointers import _

ptr = _&"hi"
print(ptr.dereference())  # prints out "hi"
```

Unfortunately, `dereference` is a pretty long name and doesn't look very pretty when you call it everywhere. Fortunately though, there are different (and more preffered) ways of dereferencing the pointer.

We can use the `_` object to once again replicate the syntax from other languages:

```py
from pointers import _

ptr = _&"hi"
print(_*ptr)
```

In some cases like this one, you can even just directly use `*`, without even having to touch `_`!

```py
ptr = _&"hi"
print(*ptr)  # works just fine
```

However, `*` is for arg splats, which introduces some problems. You can use `~` as an alternative, which will always work:

```py
ptr = _&"hi"
print(~ptr)
# ~ is a unary operator, so we can use it anywhere we want
```

## Decaying

Converting objects to pointers everywhere may be a bit ugly. To fix this, pointers.py provides a few functions to decay parameters into their pointer equivalents.

The most simple one is `decay`:

```py
from pointers import decay, Pointer

@decay
def my_function(a: str, b: str, c: Pointer):  # must be type hinted as Pointer to convert
    print(a, b, *c)

my_function('a', 'b', 'c')
```

This will be fine for most people, but it removes type safety on the target function. If you don't care about type safety in your code, then don't worry about this, but if you do, then there are alternatives.

The first alternative is `decay_annotated`, which decays parameters hinted as `Annotated[T, Pointer]` to a pointer.

Here's an example:

```py
from pointers import decay_annotated, Pointer
from typing import Annotated  # if you are below python 3.9, you can use typing_extensions here

@decay_annotated
def my_function(a: str, b: str, c: Annotated[str, Pointer]):
    print(a, b, *c)

my_function('a', 'b', 'c')
```

However, `decay_annotated` has a very large drawback.

While it adds type safety for calling the function, it breaks it inside. A type checker still thinks that the argument is a `str`, and not a `Pointer`.

Take the following as an example:

```py
@decay_annotated
def my_function(a: str, b: str, c: Annotated[str, Pointer]):
    print(a, b, ~c)  # type checker error!
```

The solution is to use `decay_wrapped`, which takes a fake function as a parameter:

```py
from pointers import decay_wrapped, Pointer

def my_function_wrapper(a: str, b: str, c: str) -> None:  # this should mimick your actual function
    ...

@decay_wrapped(my_function_wrapper)
def my_function(a: str, b: str, c: Pointer[str]):
    print(a, b, *c)
    print(a, b, ~c)  # no type checker error, it thinks c is a pointer!

my_function('a', 'b', 'c')  # works just fine, type checker things c takes a string
```

If the wrapper doesn't match, things won't work properly:

```py
from pointers import decay_wrapped, Pointer

def my_function_wrapper(a: str, b: str, c: str, d: str) -> None:
    ...

@decay_wrapped(my_function_wrapper)
def my_function(a: str, b: str, c: Pointer[str]):
    print(a, b, *c)

my_function('a', 'b', 'c')  # type checker error! missing parameter "d"
```

## Assignment

We can change where the pointer is looking at by using `assign`, or more commonly, the `>>=` operator:

```py
from pointers import _

ptr = _&"hi"
ptr.assign("hello")
ptr >>= "hello"  # equivalent to the above

print(*ptr)  # prints out "hello"
```

However, this **does not** change the original value. To do that, see the section below.

## Movement

Movement is somewhat complicated. In low level languages with pointers, you can use dereferencing assignment, like so:

```c
int b = 1;
int* a = &b;
*a = 2;
```

Unfortunately, this isn't really possible in Python. Instead, pointers.py has a feature called data movement. You can use it with `Pointer.move` or the more preffered `<<=` operator:

```py
from pointers import _

text: str = "hello world"
ptr = _&text
ptr <<= "world hello"
print(text)  # prints out "world hello"
```

**This is extremely dangerous.**

We didn't overwrite the variable `text` with `"world hello"`, we overwrote the string itself. We can run the following to then demonstrate:

```py
# ^^ insert the code from above
print("hello world")  # prints out "world hello", since we overwrote it
```

While pointers.py does its best to try and prevent segmentation faults, data movement can cause several problems, mainly with garbage collection and reference counting. If you don't know what those are, I highly recommend staying away from data movement.

In fact, unless you are familiar with the CPython internal machinery, I wouldn't touch movement at all.

### For C/C++ developers

Data movement would be like the following C code:

```c
int* ptr = &1; // lets pretend this is allowed (which it is in python)
*ptr = 2;
assert(1 == 2);
```

### Bypassing size limits

An important safety feature of movement is making sure that you can't move an object larger than the underlying value.

This is important for several reasons, but if you truly need to bypass it you can use the `^=` operator, or pass `unsafe=True` to `move`:

```py
from pointers import _

ptr = _&"123"
ptr ^= "1234" # this is the same as ptr.move("1234", unsafe=True)
```

Doing this is strictly experimental.

Moving objects too large also makes your code vulnerable to [buffer overflow attacks](https://en.wikipedia.org/wiki/Buffer_overflow), along with a chance of segmentation faults.

## Null Pointers

If you would like to point to nothing, you can use `NULL`.

Note that you cannot dereference a `NULL` pointer:

```py
from pointers import NULL, to_ptr

ptr = to_ptr(NULL)
~ptr  # NullPointerError
```

You can also assign an existing pointer to `NULL`:

```py
from pointers import NULL, to_ptr

ptr = to_ptr(1)
~ptr  # works just fine
ptr >>= NULL
~ptr  # NullPointerError
```

## Handling Segmentation Faults

If you've ever used a language like C or C++, you probably know what a segmentation fault/segfault is.

These can happen when a memory error occurs (e.g. accessing a NULL pointer), and can be annoying to debug in Python.

Luckily, pointers.py has a custom built handler for converting segfaults into Python exceptions.

Here's an example:

```py
from pointers import handle
import ctypes

@handle
def main():
    ctypes.string_at(0)  # 0 is the same as a NULL address

main()
# instead of python crashing with a segfault, pointers.SegmentViolation error occurs
```

### Pointer Methods

Most pointer methods where a segment violation could occur (`dereference`, `move`, etc.) are decorated with `handle`, so you don't have to worry about manually catching those yourself.

However, methods like `move` can be destructive and cause the error outside of the function (such as when Python does garbage collection), so you may need to make a `main` method and decorate it with `handle` to catch it.
