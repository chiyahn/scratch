
# Tutorial on interpolations.jl
This document briefly illustrates how `interpolations.jl` can be used to construct interpolations in the following scenarios:

## Interpolations on uniform grids

### Linear interpolations


```julia
using Interpolations, Base.Test
using Base.Cartesian
N = 5
xmax = 6
xmin = 1
f(x) = sin(x) / 2

# range definition
xs = linspace(xmin,xmax,N)
ys = f(xs)

# construct linear interpolation
itp = interpolate(ys, BSpline(Linear()), OnGrid())
```




    5-element interpolate(::Array{Float64,1}, BSpline(Linear()), OnGrid()) with element type Float64:
      0.420735
      0.389037
     -0.175392
     -0.499646
     -0.139708



The resulting `itp` object assumes that the data are uniformly spaced on a grid `1:N` without scaling. To scale `itp` on the grid provided (`xs`), `scale` function has to be called. Note that the grid of interest (the second parameter of `scale`) has to be given by a collection of ranges or linspaces; thus, one cannot use irregular grids for `scale` function.


```julia
# scale itp by xs
itp_scaled = scale(itp, xs)

# test if interpolation objects are in fact scaled 
dx = (xmax - xmin) / (N - 1) # distance between two grid points (assuming uniform)


# test if linear interpolations are correctly specified
@test itp[1.5] ≈ (f(xs[1]) + f(xs[2])) / 2
@test itp[2.5] ≈ (f(xs[2]) + f(xs[3])) / 2
@test itp[N-1/2] ≈ (f(xs[N]) + f(xs[N-1])) / 2

# test if linear interpolations are correctly specified, after scaling
@test itp_scaled[xmin + dx / 2] ≈ (f(xmin) + f(xmin + dx)) / 2
@test itp_scaled[xmin + 3dx / 2] ≈ (f(xmin + dx) + f(xmin + 2dx)) / 2 
@test itp_scaled[xmax - dx / 2] ≈ (f(xmax) + f(xmax - dx)) / 2 

# test if two interpolations yield identical values when evaluated at the same point upon scaling
@test itp_scaled[xmax] ≈ itp[N]
@test itp_scaled[xmin + dx / 2] ≈ itp[1.5]
@test itp_scaled[xmax - dx / 2] ≈ itp[N - .5]
```




    [1m[32mTest Passed[39m[22m



### Cubic spline interpolations
Similar techiniques can be used for cubic spline interpolation:


```julia
# construct cubic spline interpolation
itp = interpolate(ys, BSpline(Cubic(Line())), OnGrid())

# scale itp by xs
itp_scaled = scale(itp, xs)

# test if interpolation objects are in fact scaled 
dx = (xmax - xmin) / (N - 1) # distance between two grid points (assuming uniform)

# test if two interpolations yield identical values when evaluated at the same point upon scaling
@test itp_scaled[xmax] ≈ itp[N]
@test itp_scaled[xmin + dx / 2] ≈ itp[1.5]
@test itp_scaled[xmax - dx / 2] ≈ itp[N - .5]
```




    [1m[32mTest Passed[39m[22m



If it is most likely the case that the users provide grids that are not in a `1:N` format, we can have the following constructor instead:


```julia
function interp1(x,v,method)
    itp = interpolate(v, BSpline(method), OnGrid())
    return(scale(itp, x))
end

# define scaled interpolation using an alternative constructor
itp_scaled = interp1(xs,ys,Cubic(Line()))

# test if two interpolations yield identical values when evaluated at the same point upon scaling
@test itp_scaled[xmax] ≈ itp[N]
@test itp_scaled[xmin + dx / 2] ≈ itp[1.5]
@test itp_scaled[xmax - dx / 2] ≈ itp[N - .5]
```




    [1m[32mTest Passed[39m[22m



### Interpolations on multivariate data
Multivariate cases are analogous:


```julia
function interp2(x,y,v,method)
    itp = interpolate(v, BSpline(method), OnGrid())
    return(scale(itp, x, y))
end

# range definition
ymax = 3
ymin = 1
dy = (ymax - ymin) / (N - 1) # distance between two grid points (assuming uniform)
ys = linspace(ymin,ymax,N)
zs = [log(x+y) for x in xs, y in ys]

# construct cubic spline interpolations
itp = interpolate(zs, BSpline(Cubic(Line())), OnGrid())

# defined scaled interpolation using an alternative constructor
itp_scaled = interp2(xs,ys,zs,Cubic(Line()))

# test if two interpolations yield identical values when evaluated at the same point upon scaling
@test itp_scaled[xmax, ymax] ≈ itp[N, N]
@test itp_scaled[xmin + dx / 2, ymax] ≈ itp[1.5, N]
@test itp_scaled[xmax, ymin + dy / 2] ≈ itp[N, 1.5]
@test itp_scaled[xmin + dx / 2, ymin + dy / 2] ≈ itp[1.5, 1.5]
```




    [1m[32mTest Passed[39m[22m




## Interpolations on non-uniform grids


### Linear interpolations
In non-uniform grids, one can instead use gridded interpolations, which works only in linear cases:


```julia
# range definition
xs = [x^2 for x = 1:xmax]
ys = f(xs)

# construct gridded interpolation
itp = interpolate((xs,), ys, Gridded(Linear()))

# test if linear interpolations are correctly specified
@test itp[xs[1]] == f(xs[1])
@test itp[xs[2]] == f(xs[2])
@test itp[(xs[1] + xs[2]) / 2] == (f(xs[1]) + f(xs[2])) / 2
```




    [1m[32mTest Passed[39m[22m



### Cubic spline interpolations
There is no canonical way to do this, but the following approaches will be useful:

1. Construct interpolations using a recursive algorithm
    + can be extended to BSplines with arbitrary degrees
    + can be slow compared to case-by-case implementation based on degrees; requires substantial changes in code; more complexity as the degree increases; do people even use BSpline interpolations with more than 3 degrees?
    + ref: https://github.com/floswald/ApproXD.jl

2. Construct gridded interpolation algorithm dedicated for cubic splines
    + can be used to efficiently construct cubic spline interpolations
    + requires some work, but fortunately, as mentioned before, there has been a PR for this.
    + ref: https://github.com/JuliaMath/Interpolations.jl/pull/193
