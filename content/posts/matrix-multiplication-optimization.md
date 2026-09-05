+++
date = '2026-09-03T16:58:27-05:00'
draft = false
title = 'Matrix Multiplication Optimization'
tags = ['algorithms', 'optimization', 'scala']
authors = ['Luke Wassink']
+++


# Optimizing Matrix Multiplication

Outline:
- Intro: optimize matrix multiplication
    - Learn benchmarking (JMH, flame graphs)
    - Using Scala, but not functional
    - Code on GitHub
- Naive
- Flat
- Transpose
- Accumulator
- Blocks
    - slow
    - flame graph
    - nested forEach
    - fast
- Parallel
    - block size
    - thread count
- final table


Matrix multiplication is a great test case to understand performance
optimization.
The underlying algorithm is fairly simple, and it benefits measurably from some
classic optimizations like:

- reducing array access
- sequential memory access
- cache locality
- parallelization

Read this post if you want to:

- Learn some principals to make your code run faster
- Learn how to apply them to matrix multiplication (and in case it helps entice
    you, deep neural nets are basically just doing lots of matrix
    multiplication)
- Get an introduction to benchmarking your code in the JVM

This post lays out my path to a somewhat optimized matrix multiplication
algorithm.
The parameters of the project were:

- It shouldn't take to long. I spent about a week.
- Use the classic multiplication algorithm and just optimize execution. Don't
    use a fancy, complicated algorithm like XXX.
- Learn benchmarking and profiling

I've been working in Scala recently, so that's what I used.
Benchmarking and flame graphs were generated with the Java MicroBenchmarking
Harness (JMH) XXX.
You can find all the code an run it for yourself at XXX.

A disclaimer for Scala programmers: I've written plenty of pure functions in Scala. I like type
classes and monads as much as the next guy. For this project, and wanted to
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

To simplify benchmarking, we will focus on square, $n\times n$ matrices. The
product has $n^2$ entries, each of which requires $n$ operations to calculate,
so time should be $\mathcal{O}(n^3)$.

There are more sophisticated matrix multiplication algorithms such as XXX, which
preforms YYY operations, but we will not implement them. Instead, we will focus
on the simple, standard algorithm above.


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
rows (or columns, but let's stick with rows). That sounds like a list of lists,
so let's use nested arrays:

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

For an $n\times n$ matrix this is $O(n^3)$ operations. In our case $n^3 =
1,073,741,824$.

Running the benchmark, the matrix multiplication takes XXX, but we can do much
better.


## Initial improvements

Let's begin with some low hanging fruit. We are storing our matrices as nested
arrays. This adds memory overhead and extra array accesses. Instead, let's
unwind the matrix in a single, flat array with $n^2$ entries. We can access and
set the $(i, j)$ entry like so:

```scala
class FlatArray(val data: Array[Double], val rows: Int, val cols: Int):
  def apply(i: Int, j: Int): Double = data(i * cols + j)

  def set(i: Int, j: Int, d: Double): Unit = data(i * cols + j) = d
```

Multiplication stays exactly the same. This improves our runtime to XXX.

Calculating each entry requires summing $n$ doubles. Currently we accumulate the
sum in the product matrix. If we just accumulate the sum in a local variable and
set the product matrix at the end of the loop, we shave off over a billion array
accesses. This cuts our time further, down to XXX.

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
t
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

This brings the runtime down still further, to XXX, an improvement of XXX% over
our initial, naive implementation. Further progress calls for more drastic
action.


## Tiling

So far we've mostly optimized for RAM, but that's just one level of the CPU's
memory hierarchy. Each core also has an L1 cache, and accessing it can be one
hundred times faster than memory access. The CPU will try to keep recently used
data there, but if we keep asking for different data, that won't help.

Here we calculate $prod_{1,1}$ using the first row of `a` and the first column
of `b`. Then we move on the $prod_{1,2}$ and ask for the second column of `b`,
and so on. This means our data doesn't get to stick around in the cache for very
long. By the time we get to $prod_{2, 1}$ and want the first column of `b`
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
where $m = n / d$. Then we can calculate the blocks of the product by
