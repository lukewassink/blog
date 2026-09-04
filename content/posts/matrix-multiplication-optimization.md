+++
date = '2026-09-03T16:58:27-05:00'
draft = false
title = 'Matrix Multiplication Optimization'
+++

# Optimizing Matrix Multiplication

Outline:
- Intro: optimize matrix multiplication
    - Learn benchmarking (JMH, flame graphs)
    - Using scala, but not functional
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

- reducing array accesses
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

## Naive matrix multiplication

The first question: how to represent our matrices? Well, a matrix is a list of
rows (or columns, but let's stick with rows). That sounds like a list of lists,
so let's use nested arrays:

```scala
class NestedArray(val data: Array[Array[Double]])
```

