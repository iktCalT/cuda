# Lecture 8 - Data-Parallel Thinking

[Slides of Lecture 8](https://gfxcourses.stanford.edu/cs149/fall23/lecture/dataparallel/).

## Topic

![topic](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_002.jpg)

Recall that a V100 GPU has 163,840 execution contexts on one chip. We have to make sure that our program has **enough parallelism (data parallelism)** to make use of these resources.
![why data parallel](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_004.jpg)

## Key concept: dependency

Dependency is the key concept we need to understand.
![key](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_005.jpg)

So, today, instead of talking about how to **make use of parallelism** (just as we did in previous lectures), we will talk about how to write code to **create parallelism** so that the functions we wrote (or famous parallel library like NumPy) can make good use of its parallelism!

![today's goal](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_006.jpg)

## Key data type: sequences

Sequence is an ordered collection of data. Like `sequence<T>` in C++, DataFrame in Pandas and Tensor in TensorFlow and PyTorch ... But **unlike array**, you cannot use syntax like `a[i]` to access elements (recall DataFrame). This can avoid you access elements for other threads, forcing you to create a set of data **without dependency**.
![sequence](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_007.jpg)

Only a set of specific operation can access elements.

- [Map](#map)
- [Fold](#fold)
- [Scan](#scan)
- [Gather](#gather--scatter)
- [Scatter](#gather--scatter)
- [Group by key](#more-sequence-operations)
- [Filter](#more-sequence-operations)
- [Sort](#more-sequence-operations)

## Map

Map can let you access elements. It let you map a sequence of input to a sequence of output. Notice it transform a sequence (rather than one element) to another sequence.

Map takes a unary operator.
![Map](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_008.jpg)

### Parallelize Map

We can easily notice that mapping is independent. And it won't introduce any dependency. So, we can use any parallel method (like SIMD, worker thread pool ...) to implement Map!
![implement Map](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_009.jpg)

## Fold

Fold is an extremely important operation [![Fold][yt]](https://youtu.be/Ba3TqxSgnTk?t=804). Fold takes **a binary operator and a value** and folds a sequence into a single result.

And this function **takes a pair of elements** (in Map, the function takes only one element) to produce one result.
![Fold](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_010.jpg)

### \[Question\]: can you parallelize a Fold?

- My answer: it depends, if the function fulfills associative law (i.e. `f(f(a, b), c) == f(a, f(b, c))`. For example, (a + b) + c == a + (b + c)), then we can parallelize it.
- Professor's answer: yeah, function needs to be associative to be parallelized. [![can you parallelize a Fold][yt]](https://youtu.be/Ba3TqxSgnTk?t=902) (btw, function doesn't need to be commutative, just need to be associative)
![parallelize Fold as long as f is associative](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_011.jpg)

> Recall the MapReduce introduced in [Distributed System](https://youtube.com/playlist?list=PLrw6a1wE39_tb2fErI4-WkMbsvGQk9_UB). Map is Map, Reduce is Fold. -> Map and Fold are very important concepts in parallel computing.

And actually, we can fold immediately after mapping. That what JIT compiler does in PyTorch. [![map + fold][yt]](https://youtu.be/Ba3TqxSgnTk?t=1156)

## Scan

Scan takes **a binary operator**, and it converts a sequence to a sequence. It takes current input and previous output to generate current output. Here is an prefix sum example:
![Scan](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_012.jpg)

### Parallelize Scan

\*This is a huge topic.

We assume that binary function is already associative.

First of all, there are two kinds of Scan: exclusive and inclusive [![exclusive Scan and inclusive Scan][yt]](https://youtu.be/Ba3TqxSgnTk?t=1298)
![exclusive Scan and inclusive Scan](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_014.jpg)
We assume that our program uses inclusive Scan by default.

Here is an instructive discussion, watch it [![discussion][yt]](https://youtu.be/Ba3TqxSgnTk?t=1371).

![initial parallel Scan](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_015.jpg)

- Sequential: O(n)
- Parallel:
  - Work: O(n lgn)
  - Span O(lgn) time (if we have enough processors)
  - Summary: more work to do, but can be done in parallel  
  > "Span" (longest chain of sequential steps) means if you have infinite number of processors, it would take this time. "Work" means the total amount of operations to do.

This algorithm is inefficient because it has more work to do. Here is a improved version of parallel Scan:

### Work-efficient algorithm

![improved parallel Scan](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_016.jpg)
![improved parallel Scan](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_017.jpg)
It has 2 phases: combining tree (upsweep) phase and splitting tree (downsweep) phase.

- Work: O(n)
- Span: O(lgn) (more specifically: 2 * lgn)

\[Note\]: although it is efficient as first glance, it has a lot of memory operations. If you really want to make it really fast, you need to do harder.

### Exercise: parallel Scan on 2 cores

[![parallelize on 2 cores][yt]](https://youtu.be/Ba3TqxSgnTk?t=1945)

Don't need to use the algorithm above, just calculate for first and second half and add the sum of first half to next half -> 1.5x speedup.

### Exercise: parallel Scan with SIMD

Scan in a warp (recall that a [warp](../07-gpucuda/README.md/#putting-them-together) is a SIMD group and each SIMD lane in CUDA is a CUDA thread). The following graph shows how to scan 32 (we will talk about handling more elements later) elements with a 32-wide warp:
![scan_warp](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_020.jpg)

It uses only 5 (= lg32) steps. But if we change it to the [work-efficient algorithm](#work-efficient-algorithm), it takes 10 steps! [![work?][yt]](https://youtu.be/Ba3TqxSgnTk?t=2192)
![work?](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_021.jpg)

See, if you are doing things on a 2-processor machine, it's better to do simple things. If you are doing on a SIMD machine, you may choose the O(n lgn) algorithm. And if you are working on a machine with thousands of independent processors, you may choose the [advanced O(lgn) algorithm](#work-efficient-algorithm). So, if you are writing a library, it is better to write different implementation for different system.

### Building a larger scan

We can combine the methods above to build a real-world large scale scan:
![large scale](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_022.jpg)
![large scale code](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_023.jpg)

In we have an even larger array, we can divide it into blocks, and do the same thing we did above to those blocks, then combine those blocks:
![large large scale](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_024.jpg)

### Scan summary

![scan summary](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_025.jpg)

## Segmented scan

It's common to meet sequence of sequences, how to handle it?
![segmented scan](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_027.jpg)

For example: exclusive scan on sequence of sequences, you do exclusive scan to each subsequence of the outer sequence.
![segmented scan example](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_028.jpg)

Here is how to write a segmented scan code: it uses the [work-efficient algorithm](#work-efficient-algorithm) covered before. The difference is that when propagating (up-sweep), it carries the start flag information. And in the down-sweep phase, when it encounters a start flag, it will be set to 0 (initial value of scan).
![graph of segmented scan](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_030.jpg)
![code for segmented scan](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_029.jpg)

### Why we care about segmented scan

[![why segmented scan][yt]](https://youtu.be/Ba3TqxSgnTk?t=3005)

For example: sparse matrix multiplication:
![sparse matrix multiplication](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_031.jpg)
The following graph has a tiny problem, step 3 should be ${[\color{default}3x_0, \color{red}3x_0+x_2\color{default}, \color{red}2x_1\color{default}, \color{red}4x_2, \color{default}2x_1, 2x_1+6x_2, \color{red}2x_1+6x_2+8x_2\color{default}]}$
![sparse matrix multiplication](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_032.jpg)

## Gather / Scatter

These are two useful operations can be applied to sequences.

They are both data movement operations:
![gather/scatter](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_034.jpg)
![gather / scatter](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_035.jpg)

*Gather and scatter are expensive operations, because they need to jump to discrete, unpredictable places.*

There are AVX2 (SIMD) instructions supporting gather, but no instruction supports scatter:
![simd gather instructions](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_036.jpg)

### Fun trick: turn a scatter into a gather

If index array is a permutation (covers all elements and each index is unique), we can easily turn a scatter into a gather(sort) -> sort indices (elements move with indices):
![turn a scatter into a gather](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_037.jpg)

## More sequence operations

![more sequence operations](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_039.jpg)

## Why would we think in this way

Why we should think in this data parallel way and uses these operations? **Do watch this video**
[![why think in this way][yt]](https://youtu.be/Ba3TqxSgnTk?t=3827)

For fun - how to make a histogram with these concepts: [![histogram][yt]](https://youtu.be/Ba3TqxSgnTk?t=4507)

## Summary

![summary](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_050.jpg)
![summary](https://gfxcourses.stanford.edu/cs149/fall23content/media/dataparallel/images/slide_051.jpg)

<!----------- References ----------->
[yt]: https://img.shields.io/badge/YouTube-%23FF0000.svg?style=flat-square&logo=YouTube&logoColor=white
