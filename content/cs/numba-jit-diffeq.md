+++
title = "A Pattern for Accelerating Solving of Differential Equations using Numba"
date = 2023-01-28
[extra]
author = "Andy Day"
+++

## Background

The time it takes differential equation solvers (such as python's scipy LSODA) to solve a problem seems intuitive that it would be the amount of time that it takes to call the differential equation function in python plus how long it takes them to do the calculations in their own loop. Therefore, it makes sense that for a sufficiently fast equation solver, and a sufficiently complex system, that the time it takes to solve the system can be accelerated by optimizing the code for the system itself. 

## Numba: How to Make Python Go Faster

Python interpreter implementations often compile their code into python bytecode which is then run on the interpreter. However, in my experience, the bytecode is very unoptimized (loops being slow is one prime example). In contrast, Numba takes the python function and compiles it JIT using LLVM (the codegen backend for Clang, Rust, Julia, and many more). In my experience, I'm not quite sure how it works, but the compilation is *extremely* quick, in most cases; however, it seems to incur the highest penalty on the first time compiling, which indicates to me that the numba system caches things. 

One of the issues with Python code is that the type system expresses a comparatively smaller set of compile time invariants than other languages (C++, Rust, Haskell, etc), which one of many reasons optimizing python code is hard. Numba makes a few assumptions, it is numpy aware, which means it will assume the numpy datatype and optimize the hotpath for it when it compiles and will optimize for most common primitives. Every example I've tried to compile using numba has succeeded (though sometimes it falls back to what's called "object-mode", which has significant overhead as I believe that mode requires using a lot more python machinery behind the scenes, though it generally still does give a performance benefit). The hope is to have numba generate the most optimal hotpath with the smallest amount of overhead. The easiest way to do this is to pass only trivial python types (int, strings, and numpy arrays). Classes really make things difficult for numba, so avoiding those is preferred. 


### Common Pattern
```python
def diffeq(t, Y, params):
	# code here
	pass
```

When params is a python object, numba tends to use "object-mode", which is definitely unpreferable. But..., since python classes are very close type-wise to a dictionary, we the general idea is to allow this method but have it defer to a different method.

### Replacement Pattern
```python
from numba import jit


@jit(nopython=True) # <--- all that's needed for the magic!
def diffeq_numba(t, y, N, matA, matB):
	pass
def diffeq(t, y, params):
	return diffeq_numba(t, y, params.matA, params.matB)
```
Note that the jit annototation has the nopython attribute, which means that I want to assert that I never want to use object mode. Even this pattern has yielded *massive* performance gains for the differential equation system that I've tested from 12+ seconds to around 3 seconds (not a rigorous benchmark). Note that my real testing environment often defers the method call twice, for simplicity, but removing at least one layer may reduce the time further, but I have not benchmarked that.

## Caveats

Numba JIT errors are verbose and rather unhelpful. Perhaps I'm just not used to them, but they feel out of date. I had a conflict with numpy as well, which required me to regress a minor version back, which is not a big deal, just slightly annoying. It's errors also, for reasons I don't know, don't have access to the source text when running in a notebook, which is annoying because it just spits out errors with no associated source code.

## Other Benefits

Numba has additional annotations for releasing the GIL, truly parallel loops, and multithreading, utilizing all the cores on the machine. On the twelve core machine I'm writing on, the difference between being able to run one system in twelve seconds and being able to run twelve systems in four seconds by using *one library* without any major changes, well, let's just say, has it's util.


## Supplements

Environment: Jupyter-lab.

I use [poetry](https://python-poetry.org/) (highly recommend), for venv and dependencies. Here are my dependencies and versions:
```toml
python = "3.9.1"
matplotlib = "^3.7.0"
numpy = "^1.23"
jupyterlab = "^3.6.1"
scipy = "^1.10.0"
pandas = "^1.5.3"
numba = "^0.56.4"
```

