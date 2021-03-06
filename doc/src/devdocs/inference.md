# Inference

## How inference works

[Type inference](https://en.wikipedia.org/wiki/Type_inference) refers
to the process of deducing the types of later values from the types of
input values. Julia's approach to inference has been described in blog
posts
([1](https://juliacomputing.com/blog/2016/04/04/inference-convergence.html),
[2](https://juliacomputing.com/blog/2017/05/15/inference-converage2.html)).

## Debugging inference.jl

You can start a Julia session, edit `inference.jl` (for example to
insert `print` statements), and then replace `Core.Inference` in your
running session by navigating to `base/` and executing
`include("coreimg.jl")`. This trick typically leads to much faster
development than if you rebuild Julia for each change.

A convenient entry point into inference is `typeinf_code`. Here's a
demo running inference on `convert(Int, UInt(1))`:

```julia
# Get the method
atypes = Tuple{Type{Int}, UInt}  # argument types
mths = methods(convert, atypes)  # worth checking that there is only one
m = first(mths)

# Create variables needed to call `typeinf_code`
params = Core.Inference.InferenceParams(typemax(UInt))  # parameter is the world age,
                                                        #   typemax(UInt) -> most recent
sparams = Core.svec()      # this particular method doesn't have type-parameters
optimize = true            # run all inference optimizations
cached = false             # force inference to happen (do not use cached results)
Core.Inference.typeinf_code(m, atypes, sparams, optimize, cached, params)
```

If your debugging adventures require a `MethodInstance`, you can look it up by
calling `Core.Inference.code_for_method` using many of the variables above.
A `CodeInfo` object may be obtained with
```julia
# Returns the CodeInfo object for `convert(Int, ::UInt)`:
ci = (@code_typed convert(Int, UInt(1)))[1]
```

## The inlining algorithm (inline_worthy)

Much of the hardest work for inlining runs in
`inlining_pass`. However, if your question is "why didn't my function
inline?" then you will most likely be interested in `isinlineable` and
its primary callee, `inline_worthy`. `isinlineable` handles a number
of special cases (e.g., critical functions like `next` and `done`,
incorporating a bonus for functions that return tuples, etc.). The
main decision-making happens in `inline_worthy`, which returns `true`
if the function should be inlined.

`inline_worthy` implements a cost-model, where "cheap" functions get
inlined; more specifically, we inline functions if their anticipated
run-time is not large compared to the time it would take to
[issue a call](https://en.wikipedia.org/wiki/Calling_convention) to
them if they were not inlined. The cost-model is extremely simple and
ignores many important details: for example, all `for` loops are
analyzed as if they will be executed once, and the cost of an
`if...else...end` includes the summed cost of all branches. It's also
worth acknowledging that we currently lack a suite of functions
suitable for testing how well the cost model predicts the actual
run-time cost, although
[BaseBenchmarks](https://github.com/JuliaCI/BaseBenchmarks.jl)
provides a great deal of indirect information about the successes and
failures of any modification to the inlining algorithm.

The foundation of the cost-model is a lookup table, implemented in
`add_tfunc` and its callers, that assigns an estimated cost (measured
in CPU cycles) to each of Julia's intrinsic functions. These costs are
based on
[standard ranges for common architectures](http://ithare.com/wp-content/uploads/part101_infographics_v08.png)
(see
[Agner Fog's analysis](http://www.agner.org/optimize/instruction_tables.pdf)
for more detail).

We supplement this low-level lookup table with a number of special
cases. For example, an `:invoke` expression (a call for which all
input and output types were inferred in advance) is assigned a fixed
cost (currently 20 cycles). In contrast, a `:call` expression, for
functions other than intrinsics/builtins, indicates that the call will
require dynamic dispatch, in which case we assign a cost set by
`InferenceParams.inline_nonleaf_penalty` (currently set at 1000). Note
that this is not a "first-principles" estimate of the raw cost of
dynamic dispatch, but a mere heuristic indicating that dynamic
dispatch is extremely expensive.

Each statement gets analyzed for its total cost in a function called
`statement_cost`. You can run this yourself by following this example:

```julia
params = Core.Inference.InferenceParams(typemax(UInt))
# Get the CodeInfo object
ci = (@code_typed fill(3, (5, 5)))[1]  # we'll try this on the code for `fill(3, (5, 5))`
# Calculate cost of each statement
cost(stmt) = Core.Inference.statement_cost(stmt, ci, Base, params)
cst = map(cost, ci.code)
```

The output is a `Vector{Int}` holding the estimated cost of each
statement in `ci.code`.  Note that `ci` includes the consequences of
inlining callees, and consequently the costs do too.
