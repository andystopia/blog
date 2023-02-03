+++
title = "Julia: Applications in Circadian Modeling"
date = 2023-01-28
[extra]
author = "Andy Day"
+++

## Background

Historically it appears as though Python and MatLab are the established technologies utilized to model circadian clocks using differential equations. Whilst I don't have experience with Matlab, I do have experience with Python, so this review will contrast the two languages.

Julia, at the present times, appears relative destined to fall behind Python as far as popularity goes. I believe that Julia requires knowing more programming theory, or at least, less common programming theory. Writing Julia utilizes the type system to a much greater extent than python, has a type system unlike any other language I've seen, introducing a concept of "type stability". Julia variables are typed, but they are inferred, and are generally re-assignable to a new type without errors, making transferring from python slightly simpler. 

## Programmatic Theory

Julia is not object orientated. It has `struct`s, generics, and the concept of `type`s, which structs can implement, though they aren't methods directly implemented on the struct itself, more on that in a moment though.

Julia uses a system exclusively reliant on types and specialization. For anybody who has written R using S3 generics, the generic system in Julia should come quite naturally, if you're proficient with writing your own S3 generic methods. For those who have written C++ before, template specialization is a similar concept. Due to the lack of inheritance, which means no method overriding, specialization is often used. 

Due to the nature of specialization, ocassionally some sort of a code editor will be required to determine which method is being called, and from which package, and packages are free to specialize methods. 

## Ecosystem

### Differential Equations
For modeling differential equations, the library [DifferentialEquations.jl](https://docs.sciml.ai/DiffEqDocs/stable/) is *stellar*. A comparison to existing libraries can be seen below.
![Comparison Of Differential Equation Solver Software](https://camo.githubusercontent.com/97bf407cc473d22b3d9ef63c861e8dba6dd3b4579728c342c49be86b48ea180e/687474703a2f2f7777772e73746f636861737469636c6966657374796c652e636f6d2f77702d636f6e74656e742f75706c6f6164732f323031392f30382f64655f736f6c7665725f736f6674776172655f636f6d70617273696f6e2d312e706e67)

Using the differential equations package is a breeze and because Julia is compiled JIT using LLVM before executing, evaluating differential equations and situations like parameter fitting are much faster than their python counterparts with scipy. DifferentialEquations.jl contains many more solvers than Python, and the implementation of those solvers often is much faster than the Python based solutions in scipy. 

Certain uses of specialization allows for more consistent method calls when multiple differential equation systems are being used, likewise for multiple parameter sets. 

### Plotting

Julia has bindings to many plotting libraries, including an official one, Plots.jl. Plots.jl is not as fully featured as plotting libraries from other languages such as Plotly or ggplot, and I would argue that matplotlib also has more features. *That said*, the best description I can give for Plots.jl is that it feels like what matplotlib should have been. It just writes more concisely, the spacing works out rather well, and the theming appears reasonable by default. Julia also has bindings for plotly. However, that said, Pluto.jl (which I'll describe in detail later) does not support plotly and therefore is mostly limited (due to the ecosystem) to using the in built Julia Plots.jl library which is not as mature or featured as plotly.

## Where Julia Falls Short

### Maturity
Julia feels immature. I understand that it's a young language, and I will say the language documentation as well as many popular documentation is excellent, unified, and easy to navigate. 

#### Tooling
However, there are some major caveats. The Julia IDE support feels quite limited. Features such as function extraction, variable extraction, and inlining all were shaky, even when using the Julia recommended editor (VSCode) and extension.  I never had them produce anything *wrong*, just failed to activate in situations where they definitely could have been utilized. 

#### Packages

A lot of the Julia packages are very fragmented, often it will be necessary to install multiple libraries (which Pluto luckily makes easy, every notebook is its own package manager). However, it's still slightly tedious to have to install a dataframes package as well as a csv package to read in said dataframe. In general it's much less ergnomic to write in this way. Also, package imports need to be compiled by your computer for your machine the first time they're included in a project, and the Julia compiler is quite slow so it does slow down the workflow quite a bit while the packages are installing.

### Jupyter support.

Julia supports Jupyter! This is excellent for easing the transition from Python. Just set the kernel to Julia and it's all plug and play. *However*, the implementation is not without it's issues. For instance, because Julia is compiled JIT, and jupyter kernels are quite limited. Redefining a struct instance is not possible. Once a struct is implemented and the code is ran, it is *impossible* to add or remove fields from that struct. Julia will simply not allow you to recompile that cell (unless you restart the kernel, requiring recomputing all variables). To me, this is a dealbreaker. Julia is a data science language. I *should* be able to edit my code after I write it. All is not lost however, there is an excellent library called Pluto.jl which has it's own idea for notebooks. They are a single Julia source code file that is rendered and edited using a live web interface, which does seem to do something, and from what I can tell it's a form of name mangling to support structs changing definition. Pluto variables must be present in the file to be usable, and updating a variable in the editor will require a full recompute of all references that depend on that variable. I find this excellent, because there have been a fair few times where my Python notebooks have broken or worse silently wrong because of a variable used that is no longer in the notebook, but still in memory. The downside of Pluto, of course, is that the community needs to maintain it, and while it has autocomplete and live documentation (which is amazing btw), it doesn't have all the export features which Python notebooks have.


## Julia Enjoyability

In my limited experience with Julia, it feels much better to write than python. The builtin support for decent indexing without need for libraries like numpy make it much more approachable. As does the fact that packages tend to integrate relatively well and styles across libraries appear relatively consistent. 

To compare, at one point in history, I wrote a genetic algorithm in Python for parameter fitting differential equations which I re-implemented in Julia. Despite my lack of experience in Julia, my Julia implementation was done faster and ran faster, with what I would argue is feature parity. 

In general, Julia feels quite imperative rather than functional. I would argue that it feels very similar to Python as far as that balance goes. I, personally, enjoy more functional languages, and there is libraries for method piping that do tend to work quite well and I make use of them when a good use case arises. 

Julia fixes a lot of things from python. Having a stable package manager that doesn't require virtual environments makes Julia much easier to approach and programs much easier to reproduce than Python with a much lower skill required to get writing. 
