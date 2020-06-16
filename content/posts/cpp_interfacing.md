---
title: "Interfacing C++ and Python"
date: 2020-06-15T21:19:18-04:00
tags: ['python', 'c++']
---

### How to interface C++ code with Python ?

This is a quick *how-to* for :

- Installing `pybind` on WSL (which should work on Linux in general)
- Writing a small piece of code in C++
- Compiling and building the lib to expose in Python
- Using said lib.

The purpose of this is mainly to play with C++ and Python and see how the two can
be interfaced so you can benefit from the speed of C++ via Python and vice-versa.

It works both ways !

A quick list of the various libraries to do that :

- [Boost](https://www.boost.org/)
- [PyBind](https://github.com/pybind/pybind11)
- [Cython](https://cython.org/)

And probably many more but I stumbled mainly upon those while searching the internet.

I chose to go with **pybind** as it is advertised as a simplified Boost.

### 1. Installing pybind

Pre-requisites :

- python3-dev : Python dev package
- python<version_number>-dev
- [cmake](https://cmake.org/) : Tool to build, test, package software
- g++ : Compiler but that should be installed on any linux distro.

```bash
$ sudo apt-get install python3-dev cmake
```

On my first try using the library, I ran into an issue where C++ couldn't find the Python headers it needed.
So I had to specifically install the `python-dev` package that matched my Python version :

```bash
$ sudo apt-get install python3.6-dev
```

First we need to clone the repository :

```bash
$ git clone git@github.com:pybind/pybind11.git
$ cd pybind11
```

The next step is to `build` the lib.

```bash
$ mkdir build
$ cd build
$ cmake ..
$ make install
```

We also need to install the python package for `pybind`

```bash
$ pip3 install pybind11
```

After that we should pretty much be all set.

### 2. Writing a small example

This is directly and shamelessly taken from the documentation :

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <pybind11/pybind11.h>

namespace py = pybind11;

using namespace std;

int add(int i, int j) {
    return i + j;
}

PYBIND11_MODULE(example, m) {
    m.doc() = "pybind11 example plugin"; // optional module docstring

    m.def("add", &add, "A function which adds two numbers");
}
```

### 3. Compiling and building the example

To run the example this what the `pybind` doc says :

```bash
$ g++ -O3 -Wall -shared -std=c++11 -fPIC `python3 -m pybind11 --includes` example.cpp -o example`python3-config --extension-suffix`
```

The flags given here specify that we use Python3.

The `-fPIC` flag is there to specify the path to the `pybind` headers.
`pybind` comes with a sort of a convenient method to fill that out for us.

```python
$ python3 -m pybind11 --includes
-I/usr/include/python3.6m -I/usr/local/lib/python3.6/dist-packages/pybind11/include
```

Same goes for the extension of the library we are building :

```bash
$ python3-config --extension-suffix
.cpython-36m-x86_64-linux-gnu.so
```

### 4. Running the Python code

What is left to do is just to use the Python library :

```python
❯ python3
Python 3.6.9 (default, Apr 18 2020, 01:56:04)
[GCC 8.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import example
>>> help(example)
Help on module example:

NAME
    example - pybind11 example plugin

FUNCTIONS
    add(...) method of builtins.PyCapsule instance
        add(arg0: int, arg1: int) -> int

        A function which adds two numbers

FILE
    /path/to/lib/example.cpython-36m-x86_64-linux-gnu.so

>>> example.add(1, 2)
3
```

And tada we got a wonderful lib to add integers in C++ accessible from Python !

Next time we can look at building a small project example to dive more into classes, and OOP in C++
and exposing those to Python.

### Useful links

- pybind docs : https://pybind11.readthedocs.io/en/master/basics.html