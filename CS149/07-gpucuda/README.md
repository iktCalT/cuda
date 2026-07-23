# Lecture 7 - GPU architecture and CUDA Programming

Slides: [Lecture 7](https://gfxcourses.stanford.edu/cs149/fall23/lecture/gpucuda/).

## Story time

Why not listen to a story with just a click? [![story time][yt]](https://youtu.be/qQTDF0CBoxE?t=29)  

- CS 248 in 0.5 * 248 seconds: [![CS 248 in 124s][yt]](https://youtu.be/qQTDF0CBoxE?t=240).
- Abstract GPU hardware as data-parallel processor (from 3D game shader to general purpose). Read *Programming Massively Parallel Processors* (PMPP). -> CUDA came to help.

## GPU compute model

Let's review how CPU works:
![how CPU works](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_021.jpg)

How GPU works before 2007: use the 3D graphic API, treat data as if they are on triangular surfaces
![how GPU earlier works](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_022.jpg)

How GPU works after CUDA came out:
![how GPU works with CUDA](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_023.jpg)
> "kernel" in the context of CUDA means a function. `launch(myKernel, N)` means run N copies of your kernel -> SPMD

## Topics of this lecture

Think these questions carefully:
![topics](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_025.jpg)

Before we start, we need to know that CUDA thread and pthread (hardware execution context) is different. A CUDA thread is like a ISPC program instance (SIMD, vector).
![clarification](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_026.jpg)
> [!important]
> In this lecture, when we say "thread", we mean CUDA thread, rather than the thread which needs hardware context to switch.

## CUDA

Here is an example of CUDA program, just know that there is an idea called "thread block"
![thread block](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_027.jpg)

### Basic CUDA syntax

Just like ISPC, a regular C/C++ program calls a CUDA function. Note that we use terms `blockIdx`, `blockDim`, `threadIdx` to replace ISPC's `programCount`, `programIndex`, and task id:
![basic cuda syntax](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_028.jpg)

### Memory

Host: CPU
Device: GPU

GPU has its own memory and address space. So, it cannot dereference a pointer in CPU address space directly (although in modern systems, you can do so, let's just stick to traditional system for now). Instead, use `cudaMemcpy` to copy data to self address space.
![memory](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_033.jpg)

Moreover, every ***thread block* has its own address space** and every ***thread* has its own address space**
![memory 2](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_034.jpg)

So, there are:

- CPU (host) address space
- GPU (device) address space
  - global address space available to all cuda threads (**global memory**)
  - thread block address space available to all cuda threads of this block (**shared memory**)
  - thread address space only available to this thread (**local memory**)

### Example: 1D convolution

![1D convolution](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_035.jpg)
> Let's assume that input size == output size + 2, so we don't need to care about boundary condition.

Here is an easy version of solving this problem:
![1D convolution codes version 1](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_036.jpg)

Notice that there are data overlap. e.g. thread 0 reads `input[0, 1, and 2]`, and thread 1 reads `input[1, 2, and 3]`. Considering this overlap, we can write a more efficient version of code (although version 1 is not slow, thanks to caching):
![1D convolution version 2](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_037.jpg)
The `__shared__` variable in version 2 is a per-thread-block allocation (like ISPC `uniform`), this variable is shared by all threads in this block.  
On the other hand, version 1's variables and version 2's other variables (like `index` and `result`) are per-thread allocated. They are accessible only to one thread (local variable).

The reason we use this `__shared__` variable is because shared address space (available to all threads of this block) is faster than global address space (available to all threads of device).

Why we use `__syncthreads()`? -> To make sure the data needed by each thread is fetch by other threads. Otherwise, if one thread starts working, its data may still not be copied from CPU's address space [![why sync][yt]](https://youtu.be/qQTDF0CBoxE?t=2442).

### CUDA synchronization

![cuda sync](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_038.jpg)

### CUDA summary

- CUDA uses SPMD programming model. Answer to [questions](#topics-of-this-lecture): it's not dada parallel model. ([Difference between data parallel and SPMD](../04-progbasics/README.md/#two-ways-of-writing-this-program))
- What a thread does is based on its thread index, block index, ...
- They can synchronize with barriers.
- Threads are launched in bulk.
![cuda summary](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_039.jpg)

## CUDA implementation

![CUDA semantics](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_040.jpg)
> "host code" means C/C++ code running on CPU

Conceptually, we are asking GPU to create 1 million (1024\*1024) threads with 8k (1024\*1024/128) blocks. But will CUDA really do so? -> No. It is implemented like ISPC task: creating a bunch of thread blocks (similar to ISPC tasks). Then assign work to workers (cores) to do.

![assignment](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_043.jpg)

### Example: NVIDIA V100

For example: here is 1/4 of a NVIDIA V100 GPU core (here is [the whole core](#the-whole-core)). It has 16 floating point ALUs (16 yellow boxes) for SIMD calculation. And 128 sets of registers (4 warps \* 32 threads/warp sets of R0, R1, ... Actually, [later image](#the-whole-core) shows 16 warps rather than 4 warp) to store thread contexts.
![1/4 of a nv v100 core](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_045.jpg)
> R0, R1, etc are scalar registers (not vector registers).

**Although every thread has its own PC (program counter), if the several threads in the same row (called warp) share the same PC (GPU can do something to make as much thread share same PC as possible), they will run in parallel, just like SIMD**. So, at most 32 threads (in 99% of time, this happens) can run in parallel (32-wide SIMD)! It's called implicit SIMD. So, even though each register is a scalar register, 32 scalar in a row is a vector register. [![SIMD][yt]](https://youtu.be/qQTDF0CBoxE?t=3218) and [![SIMD][yt]](https://youtu.be/qQTDF0CBoxE?t=3485)
![1/4 of a nv v100 core](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_047.jpg)
![1/4 of a nv v100 core](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_048.jpg)

Notice that we have 32 threads in a warp, but we only have 16 fp32 and int ALUs. So, in the image above, a comment says "one 32-wide SIMD operation every two clocks". Also, we have only 8 fp64 ALUs. So, "one 32-wide SIMD operation every four clocks".

### Instruction scheduling

So, When the ALUs are busy, fetch/decode unit can fetch and decode instruction for other wraps. And if that wrap uses other ALUs (for example, fp32 ALUs are busy, but this warp uses fp64 ALUs), they can run in parallel! So they should look like this:

![instruction execution](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_049.jpg)

### The whole core

![a nv v100 core](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_050.jpg)

This core (also called streaming multi-processor, SM) can do 32 * 64 = 2048 CUDA threads. Every clock, it can select 4 warps (every fetch/decode choose 1 warp to run) from 64 warps to [hide latency](../03-multicore2_ispc/README.md/#hide-stalls-with-multi-threading).

It is great to assign a thread block with 2048 threads on this SM. Because all threads can use the shared memory provided by this core. But if you assign that block to a older hardware which can handle only 256 CUDA threads, your compilation may fail [![block size and implementation][yt]](https://youtu.be/qQTDF0CBoxE?t=3919). But if you run it on newer hardware, which can handle 8192 threads, there is no problem. In such case, 4 blocks will run on the same SM.

## Putting them together

Now, let's put our code and the implementation together:
![putting them together](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_051.jpg)
The hardware allocate 520 bytes shared memory for data. And uses 4 warps to hold 128 threads.

A NVIDIA V100 GPU has 80 SMs:
![a nv v100 gpu](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_053.jpg)

### Let's run our 1D program convolution on a simpler GPU

![run coevolution on simple gpu](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_055.jpg)
What's gonna happen: [![run our program][yt]](https://youtu.be/qQTDF0CBoxE?t=4221)

Notice when shared memory or execution contexts (registers for holding threads) is not enough, stop assigning work:
![work assignment](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_060.jpg)

### Why must CUDA allocate execution contexts for all threads in a block

When your block size is greater than SM's number of execution contexts, CUDA compilation will fail.

-> Because of synchronization. [![why][yt]](https://youtu.be/qQTDF0CBoxE?t=4498)
![why](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_066.jpg)
> Mentioned in [QA: Block size and implementation](#qa)

### Summary of CUDA implementation

![implementation summary](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_067.jpg)

## Supplement question

It is okay for different thread blocks to modify global memory with atomic instructions:
![okay](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_068.jpg)

But this code is not okay. Because you cannot assume the running order of blocks. If block 0 runs first. When block 1 starts, it will notice `myFlag` is 1 immediately and run normally. But if thread 1 runs first, then it will continue occupying resource until block 1 modifies `myFlag`.
![not okay](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_069.jpg)
> More importantly, if block 1 initialize `myFlag` to 0 before that while. And block 0 runs first. Then block 1 will never go out of loop.

### Bonus slides

Launch exactly as many blocks as a SM supports:
![bonus](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_070.jpg)

## Summary

![summary 1](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_071.jpg)
![summary 2](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_072.jpg)
![summary 3](https://gfxcourses.stanford.edu/cs149/fall23content/media/gpucuda/images/slide_073.jpg)

## QA

- Why use multidimensional thread id (`blockIdx`, `blockDim`, `threadIdx` have `.x` and `.y`) [![why 2D][yt]](https://youtu.be/qQTDF0CBoxE?t=1384).
- More efficient `cudaMemcpy`: it can be done asynchronously [![cudaMemcpy][yt]](https://youtu.be/qQTDF0CBoxE?t=1675).
- SPMD vs SIMD: [![spmd vs simd][yt]](https://youtu.be/qQTDF0CBoxE?t=2558)
- How [instruction are scheduled](#instruction-scheduling): [![instruction execution][yt]](https://youtu.be/qQTDF0CBoxE?t=3427)
- Thread PC and SIMD: although each thread has its own PC, GPU can do things to reconcile the PCs in a warp. So that every thread in a warp can execute same instruction at the same time. So that it can use vector instructions. [![thread pc and simd][yt]](https://youtu.be/qQTDF0CBoxE?t=3485)
- Block size and implementation: [![block size and implementation][yt]](https://youtu.be/qQTDF0CBoxE?t=3919)
- Synchronization: communication in the same SM is much faster than cross-SM communication. Try only set barriers in the same block. Because threads in the same block are in the same SM. [![synchronization][yt]](https://youtu.be/qQTDF0CBoxE?t=4119)

<!----------- References ----------->
[yt]: https://img.shields.io/badge/YouTube-%23FF0000.svg?style=flat-square&logo=YouTube&logoColor=white
