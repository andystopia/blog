+++
title = "A Pattern for Accelerating Solving of Differential Equations using Numba"
date = 2023-01-28
[extra]
author = "Andy Day"
+++

## Background

It makes sense that for a sufficiently fast equation solver, and a sufficiently complex system, that the time it takes to solve the system can be accelerated by optimizing the code for the system itself. For this, a library like [numba](https://numba.pydata.org/) which compiles python code using LLVM with some optimizations makes sense.

## Numba: How to Make Python Go Faster

Python interpreter implementations often compile their code into python bytecode which is then run on the interpreter. However, in my experience, the bytecode is very unoptimized (loops being slow is one prime example). In contrast, Numba takes the python function and compiles it JIT using LLVM (the codegen backend for Clang, Rust, Julia, and many more).

Numba tends to achieve the highest speed to simplicity ratio when using only a limited set of types (PODs and numpy ndarrays). Code that uses more complex types can make using numba more difficult. Luckily, the common patterns for differential equation solving seem to make translating code to code that works with numba relatively easy. 


### Common Pattern
```python
def diffeq(t, Y, params):
	# code here
	pass
```

When params is a python object, numba tends to use "object-mode", which is not preferable. But..., since python classes are very close type-wise to a dictionary, as are method kwargs, a simple idea is to have this method (using a python class) but have it defer to a different method, using only simple python types and numpy matricies, which can be compiled optimally.

### Replacement Pattern
```python
from numba import jit


@jit(nopython=True) # <--- all that's needed for the magic!
def diffeq_numba(t, y, N, matA, matB):
	pass
def diffeq(t, y, params):
	return diffeq_numba(t, y, params.matA, params.matB)
```
Note that the `jit` annotation has the `nopython` attribute, which means to assert that the code will build without excess python machinery. Despite the simplicity, this pattern has yielded *massive* performance gains for the differential equation system that I've tested. Reducing the time to run from 12+ seconds to around 3 seconds (not a rigorous benchmark). It would be best to not have to incur the overhead of calling the other method, but that is an undertaking where the benefits only make sense in certain use cases.

## Caveats

Numba JIT errors are verbose and rather unhelpful. Perhaps I'm just not used to them, but they feel out of date. I had a conflict with numpy as well, which required me to regress a minor version back, which is not a big deal, just slightly annoying. Its errors also, for reasons I don't know, don't have access to the source text when running in a notebook, which is annoying because it just spits out errors with no associated source code.

## Other Benefits

Numba has additional annotations for releasing the GIL, truly parallel loops, and multithreading, utilizing all the cores on the machine. Of course, Python's multithreading story is not very strong, but with proper care some subset of multithreaded programs are implementable. With  multithreading and optimizations, solving systems can be greatly accelerated. For more information on parallelism, see the `prange` function in the [numba parallelism section](https://numba.readthedocs.io/en/stable/user/parallel.html).


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

