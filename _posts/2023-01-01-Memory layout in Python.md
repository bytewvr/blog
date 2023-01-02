---
layout: post
title: Memory layout in Python
category: programming
date: 2023-01-01
tags:
  - python 
  - memory layout
  - CPython
---
I recently finished up a challenge from pwn.college where I needed to pass the address of a python bytestring (user space) to a Linux kernel module. Gdb'ing the memory occupied by python for its variables reminded me of a talk by [Brandon Rhodes - All Your Ducks In A Row: Data Structures in the Std Lib and Beyond - PyCon 2014](https://www.youtube.com/watch?v=fYlnfvKVDoM). In this post I investigate the memory layout of a few standard python objects such as `int`, `list` and `str` and the low-level structure array.
<!--more-->
# Exploring CPython's Memory Layout
Everything is an object in python and all python objects have a type and reference count. Both type and reference count are specified in the PyObject_HEAD that is part of the structure describing any object in python. According to [Python Developer's Guide - Garbage Collector Design](https://devguide.python.org/internals/garbage-collector/) a regular python object is arranged in memory as
```
object -----> +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+ \
              |                    ob_refcnt                  | |
              +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+ | PyObject_HEAD
              |                    *ob_type                   | |
              +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+ /
              |                      ...                      |
```
The fields following the header are datatype dependent and will be discussed in the following sections. To make our live easier when analyzing the memory footprint of the data types we will dump the memory content given a memory address using this python script 
```python
from ctypes import string_at
from math import ceil
import sys

def hexdump_memory(address, n_bytes):
	'''
	dump the memory starting at address in chunks of qwords (8 bytes)
	'''
    n_qwords = ceil(n_bytes / 8)
    for i in range(n_qwords):
        s = string_at(address, n_qwords * 8)[i*8:(i+1)*8]
        print(f"{int.from_bytes(s, byteorder='little'):016x}", end=' ')

        if (i+1) % 2 == 0:
            print()
```
A word of caution: the layout of memory for the data types discussed shouldn't change too often, but the following discussion is based on python version `3.10` 
# Standard datatypes
## Integers
Integers in [python 3.10](https://docs.python.org/3.10/c-api/long.html?highlight=integer) are implemented as `PyLongObjects` with arbitrary size. The declaration of the struct can be found [here](https://github.com/python/cpython/blob/3.10/Include/longintrepr.h)
Let's define an integer in python
```python
var = 1000
n_bytes = sys.getsizeof(var)
hexdump_memory(id(var), n_bytes)        
```
This leads to this memory dump, with annotations to the right
```
0000000000000001 00007f01b97a87c0  | <reference-count> <address of type>
0000000000000001 00007f01000003e8  | <obj size>        <data>
```
The second QWORD `0x00007f01b97a87c0` is the address of `class int` according to `hex(id(int))`. The 4th QWORD holds the value of the variable 0x3e8 in the lower DWORD.  `sys.getsizeof(var)` also returns a size of only 28 bytes - the smallest size for an integer variable. Owed to the unlimited size of the integer we are not limited to just 28 bytes, but can test e.g. `var = 2**120-1` which leads to the memory dump of
```
0000000000000001 00007f2e243a87c0 
0000000000000004 3fffffff3fffffff 
3fffffff3fffffff
```
The object size has increased to 4 DWORD's and the data extends across 16 bytes. 
On another note, per [documentation](https://docs.python.org/3/c-api/long.html) *When you create an int in the range between -5 and 256 you actually just get back a reference to the existing object.* Curiously, the reference-count (1st QWORD) of some integer values might reach more than a thousand references just after starting the python kernel.
```python
out = []

for i in range(-6,300):
    out.append([i, sys.getrefcount(i)])
    
out.sort(key=lambda x: x[1])
```
The first ten numbers with the highest references to them are 
```
[[0, 7605], [1, 3953], [8, 1936], [2, 1695], [12, 1159], [-1, 1137], [4, 1125], [16, 1027], [3, 978], [20, 969]]
```
The high counts of variable references are striking. 
## Strings
The `PyUnicodeObject` source code can be found [here](https://github.com/python/cpython/blob/3.10/Include/cpython/unicodeobject.h). The Unicode unique part starts after the header. I haven't seen flag structs before - the colon in the struct specifies the number of bits the element occupies. The flag struct occupies a total of 4-bytes, that seems to be QWORD aligned just before a temporary pointer that is later set to zero. 
```
typedef struct {
	PyObject_HEAD
	Py_ssize_t length; /* Number of code points in the string */
	Py_hash_t hash; /* Hash value; -1 if not set */
	struct {
		unsigned int interned:2;
...
}
```
To explore the memory footprint we allocate a 7-character long string
```python
var = 'aaaaaaa'
n_bytes = sys.getsizeof(var)
hexdump_memory(id(var), n_bytes)        
```
and get this dump
```
0000000000000001 00007fda147a3d80        | <reference-count> <address of type> 
0000000000000007 312c6e6aeb48368f        | <length>          <hash>
00007fd9ee424ae5 0000000000000000        | <state flags>     <0 address>
0061616161616161                         | <data>
```
Using C under the hood, the string is also zero terminated. This shows in the 7th QWORD that consists of 7 `0x61` with a trailing `0x00`. 
## Lists
Turning to lists,
```python
var = [1, 2, 'aaaaaaa']
n_bytes = sys.getsizeof(var)
hexdump_memory(id(var), n_bytes)        
```
we get the following dump, with annotations to the right matched to the [source code](https://github.com/python/cpython/blob/02f72b8b938e301bbaaf0142547014e074bd564c/Include/cpython/listobject.h).
```
0000000000000001 00007f2e243a8d00        | <reference-count> <address of type> 
0000000000000003 00007f2dfec3f150        | <number of elem>  <address of elem addresses>
0000000000000004                         | <allocated elems>
```
To get to the data in lists we have to follow two-levels of indirection as the object just points to a list of addresses at `0x7f2dfec3f150` that contains the addresses of the individual list elements. 
```
00007f2e23d1c0f0 00007f2e23d1c110 
00007f2e2097c6f0 0000000300006200
```
Dumping the memory at one of these three addresses gets us to the `str` or `int` records, here we go for the `int`: `hexdump_memory(0x7f2e23d1c0f0, 32)`. 
```
0000000000000f67 00007f2e243a87c0 
0000000000000001 0000000000000001
```

# Low-level data structures: array
This final section explores a low-level data structure: the [array](https://github.com/python/cpython/blob/85dd6cb6df996b1197266d1a50ecc9187a91e481/Modules/arraymodule.c). Using the array module we create an unsigned long long array with two elements. 
```python
import array
a = array.array('Q', [2**64-1, 2**64-1])
hexdump_memory(id(a), sys.getsizeof(a))
```
The dump results in
```
0000000000000001 000055b13ad5dcd0        | <reference-count> <address of type> 
0000000000000002 00007f2dfedd2ab0        | <obj size>        <address of elements>
0000000000000002 00007f2e2450aa40        | <allocated elems> <array descr>
0000000000000000 0000000000000000        | <weak refs>       <exported buffers>
```
Dumping the memory at address `0x7f2dfedd2ab0` demonstrates that array data types require one layer of indirection less than lists by specifying the numerical datatype at declaration of the array and only allowing one datatype across the array. In lists this dump would show the addresses of the elements instead of their numerical value `2**64-1 == 0xffffffffffffffff`
```
ffffffffffffffff ffffffffffffffff 
00007f2dfedd2b10 00007f2dfe9421d0 
```
# Conclusion
It was interesting to see how python handles memory under the hood. Upgraded my understanding of how objects are stored in memory and learned two completely new things: standard integer values defined at startup and C bit flag structs.
# Additional Resources
[RealPython - Memory Management in Python](https://realpython.com/python-memory-management/#the-default-python-implementation)

