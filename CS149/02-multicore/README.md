# Lecture 2 - A Modern Multi-Core Processor

You can find slides of this lecture [here](https://gfxcourses.stanford.edu/cs149/fall23/lecture/multicore/).

## Review of lecture 1

### Superscalar processors

A program is an abstraction, list of instructions saying that if you do instructions in this order, you may get this result. While superscalar is an implementation saying that we can change the order of some instructions, and by running some of them in parallel, you can get the same result but faster. [![superscalar][yt]](https://youtu.be/CKmNpAO5rS4?t=180)

## Memory

Let's continue talking about the unfinished topic of last lecture: memory.

Memory is logically an array of bytes. Caches and DRAMs are implementations of memory. Caches doesn't impact the output of a program, only influence its performance.

Fun facts: DRAMs are dynamic RAMs, it's dynamic because when loses power, data will be destroyed. Even reading the data destroys it, so, data should be written back after reading.

Cache example: [![cache example][yt]](https://youtu.be/CKmNpAO5rS4?t=755)

- Cold miss: first time access this address. It will occur even if cache is very large.
- Capacity miss: accessed this address before, but has been evicted because the working set exceeds the cache. If cache is large enough, it can be avoided.
- Conflict miss: accessed this address before, but has been evicted because the **cache set** is not large enough, even though cache might have spare space.

### Hierarchy of memory

![hierarchy of memory](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_018.jpg)

As shown below, different level of memory have significantly different access time:

![access time](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_019.jpg)

## Lecture 2's topics

Today we are going to talk 3 ideas about parallelism.

![today's topics](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_020.jpg)

### Let's see an example first

Example: [![example][yt]](https://youtu.be/CKmNpAO5rS4?t=1740)

![example](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_021.jpg)

One way to accelerate programs is using superscalar idea. Although in our example, there is no instruction level parallelism (ILP) for that in this example.

![no ILP in today's example](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_024.jpg)

### Pre multi-core era

In pre multi-core era, processors have these 3 features helping people make **single** instruction stream faster:

![pre multi-core era processors](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_025.jpg)

- Out-of-order control logic
- Fancy branch predictor
- Memory pre-fetcher

## Multi-core era processor

As the speed of single processor meets bottleneck, we can accelerate by creating **multiple** instruction steams.

Features above will accelerate the in-loop calculation of our example. But there are still tons of out-loop parallelism there. The calculation `y[i] = sin(x[i])` for each `i` is independent to each other.

## Idea 1 - use more cores

With multiple cores, even if each core is slower than single core, we can achieve faster speed if they can run in parallel.

[![idea 1][yt]](https://youtu.be/CKmNpAO5rS4?t=2053)
![idea 1](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_026.jpg)
![two cores](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_027.jpg)

Duplicating cores means duplicating everything: we have multiple fetch and decode units, multiple execution units (ALUs), multiple sets of registers, multiple caches (as well as shared caches), ... That's why they can do things in parallel.

### A simple example

But our normal code (on the right hand side) doesn't express parallelism, it will only use one core. Left hand side is the parallel (multithread) version of this function. It **explicitly (and simply) divide array `x` and `y` into two parts**, first half are handled by the new thread, while the leftover are handled by main thread. From programmer's perspective it's two threads, from machine's perspective, it's two instruction streams.

![parallel code](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_029.jpg)

### More complicated case

What if my code will run on multiple machines, and I don't know how many core each machine has?

We can declare that the loop is independent for all i from 1 to N (forall). *Specific implementation is not shown on the lecture.

## Idea 2 - Only add ALUs (SIMD)

Idea 1 asks us to use multiple cores, ALUs fetch / decode units and memory are duplicated (although multithreading doesn't create a private memory space, we are just making sure two core won't touch the same address). But SIMD tells us, sometimes, only duplicating ALUs is enough. Memory and fetch / decode units can be shared. (Btw, duplicating ALUs are much cheaper than duplicating cores.)

**SIMD**: single instruction multiple data. In our `sin(x)` example, all independent part is doing the same thing (same instructions -> SI) but handling different data (->MD). That what SIMD exactly mean.

![idea 2 SIMD](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_037.jpg)

### Vector and scalar

More importantly, the previous code handles values one by one, we call them **scalars**.

But now, we can handle multiple values at the same time (we will fetch multiple values at the same time, and then handle them in parallel). These values form a **vector**. [![vector and scalar][yt]](https://youtu.be/CKmNpAO5rS4?t=2748)

To achieve this, we can use AVX instructions. In which each time, the program will fetch a, let's say 256-bit (32 bytes = 8 * sizeof(float)) data. Which means 0 to 31 bits are `x[i]`, 32 to 63 bits are `x[i + 1]`, ... , 224 to 255 bits are `x[i + 7]`. And we can handle 8 floats at once.

![AVX instructions](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_040.jpg)

Code is pretty hard to read now. And the instructions (on the right side of figure) are used to handle vectors (`vmulps`) now, not for scalars (`vmulss`). Read **CSAPP 3.11** for more information about vector instructions.

### Combining idea 1 and 2: SIMD on multiple steams

Now notice that SIMD is running **on only one instruction stream**. And we can combine idea 2 with multiple instruction streams (idea 1). The following figure shows 16 instruction stream, each one can handle 8 floats at once. So, theoretically, 128x faster!

![combine idea 1 and 2](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_041.jpg)

### What about conditional execution?

For example,

![conditional execution](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_045.jpg)

In worst case scenario, it will be only 1/8 efficiency.

```cpp
// Pseudo code
for (int i from 0 to N) {
  if (i % 8 == 0) {do something}
  else if (i % 8 == 1) {do something}
  else if (i % 8 == 2) {do something}
  ...
}
```

Or even with one if statement

```cpp
// Pseudo code
for (int i from 0 to N) {
  if (i % 8 == 0) {
    1 million lines 
    // only useful for one 1 ALU, other 7 ALUs are waiting for it
  }
  else {
    1 line
  }
}
```

So, if you are working with SIMD on cores with many ALUs (in GPU, it can be high up to 32), be careful with conditional statements.

### Supplement

#### Coherence and divergence

- Coherence: parallel parts need same instructions.
- Divergence: parallel parts need different instructions.

Coherence is essential for SIMD.

![coherence and divergence](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_048.jpg)

#### Difference between SIMD on CPU and on GPU

They are implemented in different ways. This topic will be covered in the future.

## Summary: three forms of parallel execution

Superscalar (don't need to modify codes), SIMD, and Multi-core. [![examples][yt]](https://youtu.be/CKmNpAO5rS4?t=3885)

![three forms of parallelism](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_051.jpg)

## Idea 3 - do something else when you are waiting

![idea 3](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_062.jpg)

For example, when you meet cache miss and is waiting for data fetching from main memory, you can do something else.

![example of idea 3](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_066.jpg)

![key idea of idea 3](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_067.jpg)

[*To be continue...]

Now, we can finish [assignment 1](https://github.com/stanford-cs149/asst1).

<!----------- References ----------->
[yt]: https://img.shields.io/badge/YouTube-%23FF0000.svg?style=flat-square&logo=YouTube&logoColor=white
