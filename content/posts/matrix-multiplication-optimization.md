+++
date = '2026-09-03T16:58:27-05:00'
draft = false
title = 'Matrix Multiplication Optimization'
tags = ['algorithms', 'optimization', 'scala']
authors = ['Luke Wassink']
+++


# Optimizing Matrix Multiplication

Matrix multiplication is a great test case to understand performance
optimization.  The underlying algorithm is fairly simple, and it benefits
significantly from some classic optimizations like:

- reducing array access
- sequential memory access
- cache locality
- parallelization

There are also fancy algorithms that reduce the asymptotic complexity, but they
are so complex they usually only show benefits for very large matrices, if at
all. In any case, our plan will be to use the traditional algorithm for matrix
multiplication and explore the benefits we can get from other changes like: how
we store the matrices, what order we do the computations in, and running parts
of the computation in parallel.

I've been working in Scala recently, so that's what I used.  Benchmarking and
flame graphs were generated with the [Java Microbenchmark
Harness](https://github.com/openjdk/jmh) (JMH).
You can find all the code for this project and run it for yourself at
[github.com/lukewassink/matmult](https://github.com/lukewassink/matmult).

A disclaimer for Scala programmers: I've written plenty of pure functions in
Scala. I like type classes and monads as much as the next guy (in fact, look out
for an upcoming blog post on that very topic). For this project, I wanted to
implement classical matrix multiplication and not worry about trying to
understand the overhead of higher-order functions, so this code is imperative
and uses lots of `var`s ;)

## Matrix multiplication

Suppose we have two matrices:

$$
a = \left(
\begin{matrix}
a_{1,1} & \dots & a_{1, l} \\\\
\vdots & & \vdots \\\\
a_{n, 1} & \dots &  a_{n, l}
\end{matrix}
\right)
b = \left(
\begin{matrix}
b_{1,1} & \dots & b_{1, m} \\\\
\vdots & & \vdots \\\\
b_{l, 1} & \dots &  b_{l, m}
\end{matrix}
\right)
$$

Define $c = ab$ to be the product matrix. Then $c$ has $n$ rows and $m$ columns,
and its entries are defined by the formula

$$c_{i,j} = a_{i, 1} b_{1,j} + a_{i,2} b_{2,j} + \dots + a_{i,l} b_{l,j}.$$

To simplify the math so we can focus on optimization, this writeup will focus on
square, $n\times n$ matrices.  The code I wrote should handle matrices of any
shape, so check it out if you want details. The product of two $n\times n$
matrices has $n^2$ entries, each of which requires $\mathcal{O}(n)$ operations to calculate
for a total of $\mathcal{O}(n^3)$ operations.

There are more sophisticated approaches such as [Strassen's
algorithm](https://en.wikipedia.org/wiki/Strassen_algorithm), which only
requires $\mathcal{O}(n^{\log_2(7)})$ operations, but as mentioned above, we will
not implement them.


## Benchmarking

We'll benchmark all our algorithms against the same task: multiplying two random
1024 x 1024 matrices. All our matrix entries will be random Doubles. Scala's
`Random(seed).nextDouble` returns a value in \([0, 1)\).  That's what we'll use.

All the tests were run on my MacBook Air with:

- 16 GB memory
- Apple M2 CPU with:
    - 4 performance cores with 192 KB L1 instruction cache, 128 KB L1 data
        cache, 16 MB shared L2 cache
    - 4 efficiency cores with 128 KB L1 instruction cache, 64 KB L1 data cache,
        4 MB shared L2 cache


## Naive matrix multiplication

The first question: how to represent our matrices? Well, a matrix is a list of
rows (or columns, but let's stick with rows). This sounds like a job for nested
arrays:

```scala
class NestedArray(val data: Array[Array[Double]])
```

Let's define multiplication the simplest way we can:

```scala
for i <- 0 until a.rows do
  for j <- 0 until b.cols do
    for k <- 0 until a.cols do
      prod.set(i, j, a(i, k) * b(k, j)) // This set's the (i, j) entry of prod
```

As noted above, this is $O(n^3)$ operations. In our case $n^3 = 1,073,741,824$.
Running the benchmark, the matrix multiplication takes 1200ms, but we can do much
better.


## Initial improvements

Let's begin with some low hanging fruit. We are storing our matrices as nested
arrays. This adds memory overhead and extra array accesses. Instead, let's
unwind the matrix in a single, flat array with $n^2$ entries. We can access and
set the entres like so:

```scala
class FlatArray(val data: Array[Double], val rows: Int, val cols: Int):
  def apply(i: Int, j: Int): Double = data(i * cols + j)

  def set(i: Int, j: Int, d: Double): Unit = data(i * cols + j) = d
```

Multiplication stays exactly the same. This improves our runtime to 906ms,
already a 25% improvement.

Calculating each entry requires summing $n$ doubles. Currently we accumulate the
sum in the product matrix. If we just accumulate the sum in a local variable and
set the product matrix at the end of the loop, we shave off over a billion array
accesses. This cuts our time further, down to 865ms.

Finally, our memory access to `a` in the inner loop is nice and sequential
because rows are stored sequentially in `FlatArray`. However, `b` is not so
lucky. It's skipping through in jumps of length 1024. Sequential memory access
is faster, so we'd like to fix this. The solution is to take the transpose of
`b`:

```scala
def transpose(a: FlatArray): FlatArray =
  val t = FlatArray(a.cols, a.rows)
  for i <- 0 until a.cols do
    for j <- 0 until a.rows do
      t.set(i, j, a(j, i))
  return t
```

This along with the previous improvements, means that multiplication now looks
like:

```scala
for i <- 0 until a.rows do
  for j <- 0 until b.cols do
    var sum = 0
    for k <- 0 until a.cols do
        sum = sum + a(i, k) * bTranspose(j, k)
    prod.set(i, j, sum)
```

This brings the runtime down still further, to 831ms, an improvement of 31% over
our initial, naive implementation. Further progress calls for more drastic
action.


## Tiling

So far we've mostly optimized for RAM access, but that's just one level of the
CPU's memory hierarchy. Each core also has an L1 cache, and accessing it can be
over a hundred times faster than memory access. The CPU will try to keep recently
used data there, but if we keep asking for different data, that won't help.

The above code calculates $c_{1,1}$ using the first row of `a` and the first column
of `b`. Then it moves on the $c_{1,2}$ and asks for the second column of `b`,
and so on. This means our data doesn't get to stick around in the cache for very
long. By the time we get to $c_{2, 1}$ and want the first column of `b`
again, it's long gone from the cache.

It would be nice to do all the calculations we need on one subset of entries
from `a` and `b` all at once, so we can keep them cached. The solution is
*tiling*. Given an $n\times n$ matrix, pick a block size $d$. We'll assume $d$
divides evenly into $n$. Relaxing this is possible and doesn't change the
fundamental logic, but it does make things fiddlier. We can break our matrices
into $d\times d$ blocks:

$$
a = 
\left(
\begin{matrix}
  A_{1,1} & A_{1,2} & \dots & A_{1,m} \\\\
  A_{2,1} & A_{2,2} & \dots & A_{2,m} \\\\
  \vdots & & & \vdots \\\\
  A_{m,1} & A_{m,2} & \dots & A_{m,m}
\end{matrix}
\right),
$$

where $m = n / d$. Then we can calculate the blocks of the product by:

$$
C_{x,y} = A_{x,1}B_{1,y} + A_{x,2}B_{2,y} + \dots + A_{x,d}B_{d,y}.
$$

The products on the right side of the equation are regular matrix
multiplication. This allows us to write matrix multiplication in two steps:
first multiply the individual blocks, then multiply the matrices of blocks. For
purposes of calculating entries of $c$, this amounts to breaking our sum into an
inner and an outer sum:

$$
c_{i,j} = \sum_{x = 1}^m\sum_{k = 1}^d a_{i, xd + k}b_{xd + k, j}.
$$

We are computing the same sum-just in a different order. If we further compute
entries block-by-block rather than row-by-row, we will end up re-using the
entries of a given block in our calculation til we are done with them before
moving on to another block. For small enough blocks, this should improve our
cache locality.

We implement this algorithm as:

```scala
for i <- 0 until prod.rows by blockSize do
  for j <- 0 until prod.cols by blockSize do
    for k <- 0 until a.cols by blockSize do
      for x <- i until min(i + blockSize, prod.rows) do
        for y <- j until min(j + blockSize, prod.cols) do
          var sum = 0.0
          for z <- k until min(k + blockSize, a.cols) do
            sum = sum + a(x, z) * b(z, y)
          prod.set(x, y, prod(x, y) + sum)
```

It's not pretty, but it might be fast. The remaining question is: how big
should `blockSize` be? You could try to calculate the largest possible block
that would allow the calculation to fit in the L1 cache, but CPUs are hard to
reason about abstractly, and optimization is an experimental science. Better
just to try out some different sizes:

XXX

And... it's worse :( After looking through some flame graphs, it turns out there
are two issues:

1. Scala ranges are significantly slower when you set an increment.
1. `forEach` loops seem fine on their own, but when you nest them too deeply,
   they slow down dramatically.

To solve this, we can switch to while loops. The code is gets pretty ugly-check
out the repo if you want to see it. However, it does fix the problem:

XXX - graph

We now have an improvement of XXX% over the naive approach. This is as far as
we'll go with a single thread. Time to parallelize!


## Parallel blocks

Matrix multiplication is particularly amenable to parallelization because we can
just give each thread a different chunk of the product matrix to compute. No
shared data structures. No need for locks or mutexes.

The plan is to continue using blocks. We'll divide the rows of blocks among the
threads. For example, if we have $64\times 64$ matrix with blocks of size 8, there are 8
rows of blocks. If we use 4 threads, then each thread get's 2 rows of blocks.
That means each thread is responsible for 16 blocks, for 16 rows, or for 1024
entries, however you want to think about it. Remember we're using while loops
now. The code to compute one row of blocks is:

```scala
def computeBlockRow(i: Int): Unit =
  var j = 0
  while j < prod.cols do
    var k = 0
    while k < a.cols do
      var x = i
      while x < min(i + blockSize, prod.rows) do
        var y = j
        while y < min(j + blockSize, prod.cols) do
          var sum = 0.0
          var z = k
          while z < min(k + blockSize, a.cols) do
            sum = sum + a(x, z) * b(z, y)
            z = z + 1
          prod.set(x, y, prod(x, y) + sum)
          y = y + 1
        x = x + 1
      k = k + blockSize
    j = j + blockSize
```

Each thread will need to compute some number of rows of blocks. Let's associate
each thread with an integer `t` and write a function `computeRowsForThread(t:
Int): Unit` that fills in all the entries in the product matrix that thread `t`
is responsible for. The details are fiddly and confusing because we have to
handle the case where rows of blocks don't divide evenly among the threads;
check the repo if you're interested.

In any case, all that remains is to run each thread asynchronously in a Scala
`Future` and wait for them to fill in the results:

```scala
val futures = (0 until threadCount).map(t => Future{ computeRowsForThread(t) })
futures.foreach(Await.result(_, Duration.Inf))
```

As we've already noted, optimization is an empirical science. We shouldn't try
to guess what block size and thread count will be best. Instead, we'll just
benchmark a range of values. At first I had a bug that caused the computation to
complete correctly (thus cleverly evading the unit tests) but 
distributed the rows unevenly among the threads, causing worse performance with
a higher thread count. With the bug fixed, we get:

XXX

The best performance is for 4 threads with a block size of XXX, for a XXX%
improvement over our original, naive implementation.

To summarize, here are the benchmarks of the major versions we tried out along
the way:

XXX table
