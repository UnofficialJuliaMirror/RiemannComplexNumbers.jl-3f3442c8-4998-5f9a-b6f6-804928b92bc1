# The `RiemannComplexNumbers` module


**This module is undergoing an major reworking. Old `README` here for now.**

[![Build Status](https://travis-ci.org/scheinerman/RiemannComplexNumbers.jl.svg?branch=master)](https://travis-ci.org/scheinerman/RiemannComplexNumbers.jl)

[![Coverage Status](https://coveralls.io/repos/scheinerman/RiemannComplexNumbers.jl/badge.svg?branch=master&service=github)](https://coveralls.io/github/scheinerman/RiemannComplexNumbers.jl?branch=master)

[![codecov.io](http://codecov.io/github/scheinerman/RiemannComplexNumbers.jl/coverage.svg?branch=master)](http://codecov.io/github/scheinerman/RiemannComplexNumbers.jl?branch=master)


This is a Julia module that redefines some `Complex` operations to
give mathematically more sensible results.

## The problem we're trying to solve

Standard complex field operations in Julia work fine; the problems
begin to arise when dividing by zero. It is logical to extend the real
numbers with a positive infinity and a negative infinity. And we have
both `+Inf` and `-Inf` in Julia. However, there are problems with the
implementation of infinite values for Julia `Complex` numbers. Here
are some examples.

```julia
# For real numbers, division by 0 gives an infinite result
julia> 1/0
Inf

# This division by 0 for complex numbers is fine
julia> (2+3im)/0
Inf + Inf*im

# But this one doesn't make sense
julia> 2im/0
NaN + Inf*im

# For real numbers we have the following sensible result
julia> (Inf + 3) == (Inf + 2)
true

# But it breaks for complex numbers
julia> (Inf + 3im) == (Inf + 2im)
false
```

## This solution

For `Complex` numbers there should be just a single `Complex` infinity
(and a single `Complex` not-a-number `NaN` value). That is, we
complete the (finite) complex number system with a single additional
value at infinity.

The `RiemannComplexNumbers` module provides a value `ComplexInf` and
redefines the basic arithmetic operations to work with this
value. Also just as `0/0` yields a `NaN` value for real arithmetic, we
provide a single `ComplexNaN` value as the result of some operations.

We begin with `using RiemannComplexNumbers`. Sadly, this generates a
host of warning messages about functions being redefined (we
do not know how to suppress those warnings).

```julia
julia> using RiemannComplexNumbers
Warning: Method definition Complex(Real,Real) in module Base at
complex.jl:5 overwritten in module RiemannComplexNumbers at ...
(and so on for many, many lines)
```

But once we're past that, the examples presented above behave nicely.

```julia
julia> (2+3im)/0
ComplexInf

julia> 2im/0
ComplexInf

julia> (Inf + 3im) == (Inf + 2im)
true
```

Adding a finite value to `ComplexInf` yields `ComplexInf`, as does
multiplying `ComplexInf` by a nonzero value. However, other operations
may result in `ComplexNaN`:

```julia
julia> ComplexInf + (3-2im)
ComplexInf

julia> (5-im)*ComplexInf
ComplexInf

julia> 0*ComplexInf
ComplexNaN

julia> ComplexInf + ComplexInf
ComplexNaN
```

The last example deserves a bit of explanation. For real arithmetic
`Inf-Inf` yields `NaN`, but `Inf+Inf` yields `Inf`. This is
sensible. But for `Complex` values with a single `ComplexInf`, there
is no difference between adding or subtracting two `ComplexInf`
values. (This is the same behavior as in *Mathematica*.)

Note that division by infinity yields zero as expected:

```julia
julia> im/ComplexInf
0.0 + 0.0im
```

### Note

The `RiemannComplexNumbers` module does not define a new `Complex`
type; it only redefines basic operations to give sensible results. At
this writing, the redefined division operator is not as carefully
implemented as the original; this should be fixed.

The redefined arithmetic operations are less efficient than the
originals as various checks are performed to ensure consistent
results.

The value `ComplexInf` is represented internally as `Inf + Inf*im` but
the `show` function has been redefined to print this as
`ComplexInf`. Likewise for `ComplexNaN`.

`Complex` numbers in matrices are written in a compact form `3+2im` as
opposed to `3 + 2im`. The compact forms for `ComplexInf` and
`ComplexNaN` are `C_Inf` and `C_NaN`.

## ~~Problems with complex division~~

### ~~Inaccuracies~~

**This problem has been fixed** (I think).

I copied code from `base.complex.jl` that handles complex division and
incorporated that into my methods. So now `1/0.2im` returns
`0.0 - 0.5im` as desired.

---
**Description of the (now solved) problem**


Our current reimplementation of complex floating point arithmetic,
even for finite values, is not the same as the functions distributed
with Julia and need to be updated. In particular, Julia's division
handles floating point values more accurately.  Observe:

```julia
julia> z = (1/5)*im
0.0 + 0.2im

julia> 1/z
0.0 - 5.0im

julia> using RiemannComplexNumbers
Warning: Method definition Complex(Real,Real) in ...
(and so on for many lines)

julia> 1/z
0.0 - 4.999999999999999im
```
---


### ~~Our division method doesn't always get executed~~

**This problem has been fixed.**  My confidence in this assertion is not
overwhelming. I think the fixes I put in place take care of all situations,
but I'm not 100% sure.


---
*Description of this now-repaired  bug*

Division of a nonzero `Complex` value by zero should give
`ComplexInf`, as in this example:
```julia
julia> (1+0im)/(0+0im)
ComplexInf
```

However, when we divide these as floating point complex numbers, we
run into trouble:
```julia
julia> (1.0+0.0im)/(0.0+0.0im)
ComplexNaN
```
The answer, of course, should be `ComplexInf`.

Somehow, the integer version gets routed to the correct method, but
the floating point version does not:
```julia
julia> @which (1+0im)/(0+0im)
/(w::Complex{T<:Real},z::Complex{T<:Real}) at /home/ers/.julia/v0.3/RiemannComplexNumbers/src/RiemannComplexNumbers.jl:122

julia> @which (1.0+0.0im)/(0.0+0.0im)
/(z::Complex{Float64},w::Complex{Float64}) at complex.jl:162
```

Thanks to @pearlzli for pointing this out. ~~Anyone know how to fix this??~~

---




## An alternative solution

It would be possible, and perhaps desirable, to create an alternative
`Complex` type (say, `RiemannComplex`) that is simply a wrapper around
an ordinary `Complex` value. Operations on `RiemannComplex` numbers
would have their own definitions (which would rely on standard
definitions when all values are finite). We opted not to go that route
because that would not be useful to existing code that uses `Complex`
numbers.
