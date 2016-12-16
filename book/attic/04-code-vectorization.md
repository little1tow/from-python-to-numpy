# Chapter 4 - Code vectorization

## Introduction

Code vectorization means that the problem  you're trying to solve is inherently
vectorizable  and only  requires a  few (numpy)  tricks to  make it  faster. Of
course, it does  not mean it is  easy nor straighforward, but at  least it does
not necessitate  to totally rethink your  problem.  Still, it may  require some
experience to see  where and how code can be  vectorized. Let's illustrate this
through  a  very  simple  example  where  we  want  to  sum  up  two  lists  of
integers. One simple way using pure Python is:

```Python
def add_python(Z1, Z2):
   return [z1+z2 for (z1,z2) in zip(Z1,Z2)]
```

This first naive solution can be immediately vectorized as:

```Python
def add_numpy(Z1, Z2):
    return np.add(Z1, Z2)
```
    
Note that we did not write `Z1 + Z2` because it would not work if `Z1` and `Z2`
were both lists.  Without any surprise, benchmarkming the  two approaches shows
the second method is the fastest with almotst two orders of magnitude.

```Pycon
>>> Z1 = random.sample(range(1000), 100)
>>> Z2 = random.sample(range(1000), 100)
>>> timeit("add_python(Z1, Z2)", globals())
1000 loops, best of 3: 68 usec per loop
>>> timeit("add_numpy(Z1, Z2)", globals())
10000 loops, best of 3: 1.14 usec per loop
```
    
Not only  the second approach  is faster, but it  also naturally adapts  to the
nature and  the shape of  `Z1` and `Z2`,  which is not  the case for  the first
method because of  the inner `+` which is interpreted  differently depending on
the nature  of the  two objects.  For example,  if we  now consider  two nested
lists:

```Pycon
>>> Z1 = [[1, 2], [3, 4]]
>>> Z2 = [[5, 6], [7, 8]]
>>> Z1 + Z2
[[1, 2], [3, 4],[5, 6], [7, 8]]
>>> add_python(Z1, Z2)
[[1, 2, 5, 6], [3, 4, 7, 8]]
>>> add_numpy(Z1, Z2)
[[ 6  8]
 [10 12]]
```

The  first  method concatenates  the  two  lists  together, the  second  method
concatenates the  internal lists  together and  the last  one computes  what is
(numerically) expected. 



## Differential vectorization

The Mandelbrot set is the set of complex numbers `c` for which the function
`fc(z) = z²+ c` does not diverge when iterated from z=0, i.e., for which the
sequence fc(0), fc(fc(0)), etc., remains bounded in absolute value. It is very
easy to compute but it can take a very long time because you need to ensure a
given number does not diverge. This is generally done by iterating the
computation up to a maximum number of iterations, after which, if the number is
still within some bounds, it is considerer non divergent. Of course, the more
iteration, the more precision. A pure python implementation is written as:

```Python
def mandelbrot_1(xmin, xmax, ymin, ymax, xn, yn, maxiter, horizon=2.0):
    def mandelbrot(z, maxiter):
        c = z
        for n in range(maxiter):
            if abs(z) > horizon:
                return n
            z = z*z + c
        return maxiter
    r1 = [xmin+i*(xmax-xmin)/xn for i in range(xn)]
    r2 = [ymin+i*(ymax-ymin)/yn for i in range(yn)]
    return [mandelbrot(complex(r, i),maxiter) for r in r1 for i in r2]
```

The interesting (and slow) part of this code is the `mandelbrot` function that
actually computes the sequence fc(fc(fc ...))). The vectorization of such code
is not totally straighforward because the internal `return` implies a
differential processing of the element. Once it has diverged, we don't need to
iterate any more and we can safely return the iteration count at
dievergence. Problem is then to do the same numpy. But how ?

The trick is to search at each iteration values that have not yet diverged and
update relevant information for these values and only these values. Because we
start from Z = 0, we know that each value will be updated at least once (when
they're equal to 0, they have not yet diverged) and will stop being updated as
soon as they've diverged. To do that, we'll use numpy fancy indexing with the
`less(x1,x2)` function that return the truth value of (x1 < x2) element-wise.

```Python
def mandelbrot_2(xmin, xmax, ymin, ymax, xn, yn, maxiter, horizon=2.0):
    X = np.linspace(xmin, xmax, xn, dtype=np.float32)
    Y = np.linspace(ymin, ymax, yn, dtype=np.float32)
    C = X + Y[:,None]*1j
    N = np.zeros(C.shape, dtype=int)
    Z = np.zeros(C.shape, np.complex64)
    for n in range(maxiter):
        I = np.less(abs(Z), horizon)
        N[I] = n
        Z[I] = Z[I]**2 + C[I]
    N[N == maxiter-1] = 0
    return Z, N
```

Here is the benchmark:

```Pycon
>>> xmin, xmax, xn = -2.25, +0.75, int(3000/3)
>>> ymin, ymax, yn = -1.25, +1.25, int(2500/3)
>>> maxiter = 200

>>> print_timeit("mandelbrot_1(xmin, xmax, ymin, ymax, xn, yn, maxiter)", globals())
1 loops, best of 3: 6.1 sec per loop
>>> print_timeit("mandelbrot_2(xmin, xmax, ymin, ymax, xn, yn, maxiter)", globals())
1 loops, best of 3: 1.15 sec per loop
```

There gain is roughly a x5 factor, not as much as we can have expected. Part of
the problem is that the `np.less` function implies `xn*yn` tests at every
iteration while we know that some values have already diverged. Even if these
tests are performed at the C level (through numpy), the cost is nonetheless non
negligible. Another approach proposed
by [Dan Goodman](https://thesamovar.wordpress.com/) is to work on a dynamic
array at each iteration that stores only the points which have not yet
diverged. It requires more lines but the result is faster and lead to a 10x
factor speed improvement compared to the Python version.

```Python
def mandelbrot_3(xmin, xmax, ymin, ymax, xn, yn, itermax, horizon=2.0):
    Xi, Yi = np.mgrid[0:xn, 0:yn]
    Xi, Yi = Xi.astype(np.uint32), Yi.astype(np.uint32)
    X = np.linspace(xmin, xmax, xn, dtype=np.float32)[Xi]
    Y = np.linspace(ymin, ymax, yn, dtype=np.float32)[Yi]
    C = X + Y*1j
    N_ = np.zeros(C.shape, dtype=np.uint32)
    Z_ = np.zeros(C.shape, dtype=np.complex64)
    Xi.shape = Yi.shape = C.shape = xn*yn

    Z = np.zeros(C.shape, np.complex64)
    for i in range(itermax):
        if not len(Z): break

        # Compute for relevant points only
        np.multiply(Z, Z, Z)
        np.add(Z, C, Z)

        # Failed convergence
        I = abs(Z) > horizon
        N_[Xi[I], Yi[I]] = i+1
        Z_[Xi[I], Yi[I]] = Z[I]

        # Keep going with those who have not diverged yet
        np.negative(I,I)
        Z = Z[I]
        Xi, Yi = Xi[I], Yi[I]
        C = C[I]
    return Z_.T, N_.T
```

Benchmark gives us:

```Pycon
>>> print_timeit("mandelbrot_3(xmin, xmax, ymin, ymax, xn, yn, maxiter)", globals())
1 loops, best of 3: 510 msec per loop
```

Here is a picture of the result where we use recount normalization, power
normalized colormap (gamma=0.3) and shading. See [mandelbrot.py)

![](../pics/mandelbrot.png)

We now want to measure the fractal dimension of the Mandelbrot set using
the
[Minkowski–Bouligand dimension](https://en.wikipedia.org/wiki/Minkowski–Bouligand_dimension). To
do that we need to do a box-counting with decreasing box size. As you imagine,
we cannot use pure Python because it would be way to slow. Instead, we'll use
the
[numpy.ufunc.reduceat](https://docs.scipy.org/doc/numpy/reference/generated/numpy.ufunc.reduceat.html) that
allows to combine a ufunc function over a local reduce with specified slices
over a single axis.



```Python
def fractal_dimension(Z, threshold=0.9):
    def boxcount(Z, k):
        S = np.add.reduceat(
            np.add.reduceat(Z, np.arange(0, Z.shape[0], k), axis=0),
                               np.arange(0, Z.shape[1], k), axis=1)
        return len(np.where((S > 0) & (S < k*k))[0])

    Z = (Z < threshold)
    p = min(Z.shape)
    n = 2**np.floor(np.log(p)/np.log(2))
    n = int(np.log(n)/np.log(2))
    sizes = 2**np.arange(n, 1, -1)
    counts = []
    for size in sizes:
        counts.append(boxcount(Z, size))
    coeffs = np.polyfit(np.log(sizes), np.log(counts), 1)
    return -coeffs[0]
```



**Code**

  * [mandelbrot.py](../code/mandelbrot.py)

**References**

* [How To Quickly Compute the Mandelbrot Set in Python](https://www.ibm.com/developerworks/community/blogs/jfp/entry/How_To_Compute_Mandelbrodt_Set_Quickly?lang=en)  
  Jean Francois Puget, 2015.
* [My Christmas Gift: Mandelbrot Set Computation In Python](https://www.ibm.com/developerworks/community/blogs/jfp/entry/My_Christmas_Gift?lang=en)  
  Jean Francois Puget, 2015.
* [Fast fractals with Python and Numpy](https://thesamovar.wordpress.com/2009/03/22/fast-fractals-with-python-and-numpy/)  
  Dan Goodman, 2009.
* [Renormalizing the Mandelbrot Escape](http://linas.org/art-gallery/escape/escape.html)  
  Linas Vepstas, 1997.

## Localized vectorization
## Coupled vectorisation
## Earthquake visualization
## Conclusion