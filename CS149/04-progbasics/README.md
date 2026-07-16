# Lecture 4 - Parallel Programming Basics

This lecture include slides from [lecture 3](https://gfxcourses.stanford.edu/cs149/fall23/lecture/multicore2-ispc/) and [lecture 4](https://gfxcourses.stanford.edu/cs149/fall23/lecture/progbasics/).

## Review

4 concepts are introduced by the professor:

- Multicore
- SIMD
- Multithreading
- Superscalar

Assignment 1 is conceptual assignment, these concepts are practiced one by one. But when you are writing programming assignments on real-world hardware, things will be messier. Because these things will occur together. So, you may doubt your understanding when facing real-world program. But do trust yourself, your understanding is correct, it's just because things are mixed up.

## Implementations of ISPC gangs

Last lecture, we've talked about abstractions of ISPC gangs, now, let's look at its implementations.

ISPC gang is implemented exactly the same as what we did in Assignment 1 – Part 2: **SIMD**.

![implementation of ISPC gangs](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore2-ispc/images/slide_036.jpg)

So, when your main function calls a ISPC function, you will notice that the thread running from scalar instructions to vector instructions. And when the ISPC function returns, it changes back to scalar instructions. The whole things happen on 1 thread.

### Interleaved vs blocked assignment

Interleaved assignment is more efficient than blocked. Because when you fetch SIMD data (a vector), all members are in the same cache line (if vector size <= cache line size). So, *interleaved assignment is more efficient than blocked assignment*!

[![interleaved vs blocked assignment][yt]](https://youtu.be/0-ztm8SKq70?t=794)

### ISPC `foreach`

`foreach` let ISPC to figure out how to map different program instances to each iteration. Let ISPC to decide with assignment (interleaved, blocked, or something else) to use.

![ISPC foreach](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore2-ispc/images/slide_042.jpg)

\[Note\] `foreach` may choose a bad assignment (but usually, it does good jobs). So it also offers both higher-level `foreach` and lower-level `programCount` and `programIndex`. You can write your version of implementations with these lower-level tools.

From now on, please think in a `foreach` way—think what are the jobs to be done in each iteration (remember that in Java `foreach` means iterate through), don't think of instances and gangs.

### Avoid race condition

Although `foreach` makes it possible for us to forget instances and gangs, do keep in mind that we are still doing parallelization. So, each iteration must run **independently**!!!

This is an invalid ISPC program, iteration i and iteration i - 1 are dependent. We hope we know `y[9]` before running `y[9] = x[10]`, but ISPC does NOT guarantee that i = 9 runs before i = 10!
![an invalid example](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore2-ispc/images/slide_045.jpg)

If you did something wrong with that, ISPC compiler cannot point out for you. [![QA][yt]](https://youtu.be/0-ztm8SKq70?t=1353)

### Uniform and how to handle with race condition

A normal type (e.g. `float`) has one copy for each instance. While a uniform type (e.g. `uniform float`) has only one copy.

Left side program cannot run because it doesn't know which copy of `sum` should return. Right side program cannot run, too, because of race condition.
![uniform](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore2-ispc/images/slide_046.jpg)

Here is a correct version of this program:
![correct version](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore2-ispc/images/slide_047.jpg)
`reduce_add` is a function provided by ISPC. It is implemented with vector load (`_mm256_load_ps`) and vector add(`_mm256_add_ps`).

#### Other cross-instance functions

![cross-instance functions](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore2-ispc/images/slide_048.jpg)

But if you want to write a custom cross-instance lambda function like `reduce_add`...  No such a way to do so for ISPC (so far).

## ISPC tasks

The topics we have talked so far are still happening on only one core and one thread. We can replicate them on multiple threads—with the help of ISPC tasks.

![ISPC tasks](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore2-ispc/images/slide_051.jpg)

- ISPC foreach: here is a gazillion loop iterations, you (ISPC) will figure out how to assign it to program instances.
- ISPC tasks: here is a gazillion gangs, you figure out how to assign it to the threads of machine.

If you want to know how to write a program with ISPC tasks, see [Assignment 1 Program 3](https://github.com/stanford-cs149/asst1#program-3-parallel-fractal-generation-using-ispc-20-points).

## ISPC is also a low-level language

Another way why ISPC provides lower-level things like `programCount` and `programIndex` is we can use them to do advanced stuffs.

![advanced stuff with low-level feature](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore2-ispc/images/slide_053.jpg)

If we remove those lower-level stuffs in ISPC, it is like a higher-level language, like Numpy or PyTorch.

## ISPC summary

![summary](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore2-ispc/images/slide_056.jpg)

## Parallel Programming Basics

Now comes to the lecture 4's topics. The speedup we got in previous classes is not super crazy. Now, we are going to reduce the timer by several factor.

![steps for creating a parallel program](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_004.jpg)

### Step 1: decomposition

The first step is always look at the problem and figure out how to decompose it.
![decompose problem](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_005.jpg)

But Amdahl's law tells us that if you only speedup one part of the program, there is a limit (this is easy to understand).
![Amdahl's law](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_006.jpg)

For example:

![example part 1](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_007.jpg)
![example part 2](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_008.jpg)

What we can do is speed up all time consuming parts of our program. Here is a tail in this example, because we need to use `reduce_add` to add up all partial sum.

![example solution](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_009.jpg)

And here is a graph of P-MaxSpeedup. Where s represent the ratio of work cannot be parallelized.
![alt](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_010.jpg)
As shown in this graph, if s is greater (sequential parts are large), you will face limit when you have less processors. If you are using a large parallel machine, you need a very very small s (maybe 0.00...001) to make sure it doesn't hit the wall.
![hazard of sequential parts](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_011.jpg)

So, decomposition is very important. And usually, programmers are responsible for decomposition. Auto decomposition is still impossible so far.
![programmers should be responsible for decomposition](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_012.jpg)

### Step 2: Assignment

**The key of assignment is to keep all of my workers busy**.
![assignment](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_014.jpg)

Assignment is the part that we can rely on compilers to do for us (and usually, they do better than us). For example, we use ISPC `foreach` and ISPC tasks to assign works. But we can also manually assign works (like what we did in assignment 1, program 1).

### Step 3: Orchestration

For example, synchronization and `reduce_add`.
![orchestration](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_019.jpg)

### Mapping

We have seen several:
![mapping](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_021.jpg)

## A parallel program example

![parallel program example](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_023.jpg)
Recommend to watch the video [![parallel program example][yt]](https://youtu.be/0-ztm8SKq70?t=3218)

### Decomposition

Here is a pseudocode, but it cannot be paralleled, because a point is dependent to adjacent points.
![pseudocode](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_024.jpg)

Although you can achieve some independence for points in the same [diagonal](https://gfxcourses.stanford.edu/cs149/fall23/lecture/progbasics/slide_26). But we won't do it, because 1. it's not a large amount of independence, 2. it has poor spacial locality.

A good way to solve this problem is to choose another algorithm—red-black coloring. This may not converge at the same value as sequential method, but it will fulfill the threshold in the end.
![change the algorithm](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_027.jpg)
![new algorithm](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_028.jpg)

### Choose an assignment

Which one is better?
![assignment: which one is better](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_029.jpg)

Consider dependency
![assignment:](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_030.jpg)

Blocked assignment needs less synchronization and less inter-process communication (and considering data fetching). So, blocked assignment is better.

### Two ways of writing this program

![two ways](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_032.jpg)

- Data parallel way: tell the system that I have done the decomposition, and here are the things you need to finish. Here is the pseudocode (forget ISPC for now)
  ![data parallel way](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_034.jpg)

- Shared address space (with SPMD threads) way: think of **a bunch threads running in a shared address space, but we need to synchronize sometime (with locks and barriers)**.
  ![shared address space + SPMD way](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_036.jpg)
  ![shared address space + SPMD implementation](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_037.jpg)
  \*Orchestration is done by lock.

To choose the assignment method by ourself, we need to use the second way, a lower-level way. But there is a problem with shared address space + SPMD—**race condition**.

### What does lock do in shared address space

**Lock (here, it refers to mutex lock)** can make sure things won't go wrong due to **race condition** in a **shared address space**. **Atomic instructions (provided by hardware or some languages)** can also solve this problem.
![race condition](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_039.jpg)
![solve race condition](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_043.jpg)

But synchronization (caused by locks) also cause performance problem. So, do the synchronization as infrequently as possible For example, don't `lock(myLock); diff += myDiff; unlock(myLock);` in the `for (j=myMin to myMax){}` loop, do it outside this loop!

### What does barrier do in shared address space

Barrier is another form of synchronization. Threads are blocked here unless NUM_PROCESSES (all) threads reaches here (like `for(thread in threads) thread.join();`)
![barrier](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_047.jpg)

Why we need 3 barriers in the example?
![3 barriers](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_048.jpg)

- The second one is easy: we need to add all `myDiff` to `diff`, so that we can determine whether work is done or not.
- The third one: for example, if we don't have second barrier. When thread 3 has finished comparison (`if (diff/(n*n) < TOLERANCE)`), and it goes into next loop, then it set `diff = 0.f`. But now, thread 2 have not started comparison yet. Then, when thread 2 start comparing, it find that `diff/(n*n)` is 0 (`< TOLERANCE`), and thread 2 quits in the next loop.
- The first barrier: if we don't have it. When thread 3 has already modified `diff` by `diff += myDiff`. After that, thread 2 set `diff = 0.f`, then `diff` is not correct.

\[Challenge\] [![challenge][yt]](https://youtu.be/0-ztm8SKq70?t=4564) Can you use only one barrier? Don't use the same copy of `diff` for every iteration.  -> Answer: [Lecture 5](../05-perfopt1/README.md).

### Summary of this example

![summary](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_050.jpg)

## Questions & supplements

### About ISPC gang size

How does it determined? You set it at compile time. [![how does gang size determined][yt]](https://youtu.be/0-ztm8SKq70?t=374)

Although it is implemented in SIMD, if you set gang size > SIMD width, the program can still run correctly. If you set gang size to be 664, but your PC supports only 8 SIMD lanes, ISPC will issue 8 instructions (on one thread) to do what could be done with 1 instruction. But usually, you will get better performance (because it has larger ILP). [![when you assign a large gang size][yt]](https://youtu.be/0-ztm8SKq70?t=874)

### Tasks, threads, and context switch

Two good questions: 1. tasks vs threads; 2. context switch (context switch has cost, watch the video). [![QA][yt]](https://youtu.be/0-ztm8SKq70?t=1900)

> In CPU specifications, 8 core 16 threads means, it can run at most 16 thread in parallel. But your program can spawn many many threads.

We assume that our program is the only program running on this machine. So, it makes no sense for our application to create more threads than execution context.

But, don't mix up the OS level context switch and hardware-level multithreading (also context switch). OS-level context switch manages which core runs which thread. Hardware multithreading means switch instruction stream to hide stall. A OS-level context switch happens in hundreds of cycles, however, a hardware-level context switch happens in one cycle (it only changes PC). So, **when we are talking about using multithreading to hide stall, don't think of OS-level context switch.**

<!----------- References ----------->
[yt]: https://img.shields.io/badge/YouTube-%23FF0000.svg?style=flat-square&logo=YouTube&logoColor=white
