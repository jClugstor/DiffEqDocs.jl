# [I/O: Saving and Loading Solution Data](@id io)

The ability to save and load solutions is important for handling large datasets
and analyzing the results over multiple Julia sessions. This page explains the
existing functionality for doing so.

## Tabular Data: IterableTables

An interface to [IterableTables.jl](https://github.com/queryverse/IterableTables.jl)
is provided. This IterableTables link allows you to use a solution
type as the data source to convert to other tabular data formats. For example,
let's solve a 4x2 system of ODEs and get the DataFrame:

```@example IO
using OrdinaryDiffEq, DataFrames
f_2dlinear = (du, u, p, t) -> du .= 1.01u;
tspan = (0.0, 1.0)
prob = ODEProblem(f_2dlinear, rand(2, 2), tspan);
sol = solve(prob, Euler(); dt = 1 // 2^(4));
df = DataFrame(sol)
```

If we set `syms` in the DiffEqFunction, then those names will be used:

```@example IO
f = ODEFunction(f_2dlinear, syms = [:a, :b, :c, :d])
prob = ODEProblem(f, rand(2, 2), (0.0, 1.0));
sol = solve(prob, Euler(); dt = 1 // 2^(4));
df = DataFrame(sol)
```

Many modeling frameworks will automatically set `syms` for this feature.
Additionally, this data can be saved to a CSV:

```@example IO
using CSV
CSV.write("out.csv", df)
```

## JLD2 and BSON.jl
Solution objects contain function types, which can cause JLD2 and BSON.jl to serialize and deserialize incorrectly. 
JLD2.jl will work correctly on solution objects only after `SciMLBase.strip_solution` is called on the object. 
For example, if we save the stripped solution:

```@example IO
sol = solve(prob, Euler(); dt = 1 // 2^(4))
stripped_sol = SciMLBase.strip_solution(sol)
using JLD2
@save "out.jld2" stripped_sol
```
then we can get the stripped solution object back, interpolations intact.

```@example IO
# New session
using JLD2
using OrdinaryDiffEq
loaded_sol = load("out.jld2") 
```
Note that stripped solutions do not contain any problem information. 

BSON.jl seems to work better with some functions, so it might not be necessary to
strip the solution. Of course a stripped solution will serialize and deserialize 
with BSON.jl just as well. 

```@example IO
sol = solve(prob, Euler(); dt = 1 // 2^(4))
using BSON
bson("test.bson", Dict(:sol => sol))
```

```@example IO
# New session
using OrdinaryDiffEq
using BSON
BSON.load("test.bson")
```

For all algorithms you'll need
at least a solver package or SciMLBase.jl in scope in order for
the solution interface (plot recipes, array indexing, etc.) to
work. If none of these are put into scope, the solution type
will still load and hold all of the values (so `sol.u` and `sol.t`
will work), but none of the interface will be available.

## JLD

Don't use JLD. It's dead. Julia types can be saved via JLD.jl.
However, they cannot save types which have functions, which means that
the solution type is currently not compatible with JLD.

```julia
using JLD
JLD.save("out.jld", "sol", sol)
```
