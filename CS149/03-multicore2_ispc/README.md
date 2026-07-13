# Lecture 3 - Multi-core Arch Part II + ISPC Programming Abstractions

You can find slides of this lecture from [lecture 2](https://gfxcourses.stanford.edu/cs149/fall23/lecture/multicore/) and [lecture 3](https://gfxcourses.stanford.edu/cs149/fall23/lecture/multicore2-ispc/).

## Review

- Multi-core execution
- SIMD execution
- Hardware multi-threading

## Hide stalls with multi-threading

Last topic from last lecture: when you are waiting for something (for example, waiting for data from main memory), do something else. This makes our use of core more efficiently. -> We can hide stalls with multi-threading.

![hiding stalls with multi-threading](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_066.jpg)

In the image above, 1 set of instructions, 4 threads, 100% CPU utilization.

### The cost of this idea

- We need larger space to hold these execution contexts. So, if we creates too many threads, the space for storing threads' states will be very small.
- Each thread takes more time to be finished. Because it shares core with other threads. If it finishes waiting and can be processed, it may need to wait other thread to stop occupying the core.

![cost of multi-threading](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_067.jpg)

### Hardware supporting this idea

Some hardwares have the ability to maintain the states of two threads, it's called *multithreaded hardware*, and it supports *hardware multithreading*.

Here is an example — a core can hold the state of 2 instruction streams (threads):

![hardware multithreading](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_071.jpg)

When one thread stalls, it can switch to another quickly.

### When we create more and more threads

![1 thread](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_072.jpg)
![5 threads](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_075.jpg)
![more threads](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_076.jpg)

If we add more threads, we will get higher efficiency at the beginning, but when the utilization gets 100%, it won't be faster. Moreover, we need more space for contexts and more time for scheduling and switching.

> Switching form one thread to another is actually pretty fast. It's just switching form a PC (program counter) to another PC.

The number of threads needed to get 100% utilization is determined by the ratio of arithmetic and memory-load latency.

> Big data cache can reduce the probability of cache miss, and reduce the average memory load latency! So, we need less threads to achieve 100% utilization (although it won't make the program faster).

![takeaway 1](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_079.jpg)
![takeaway 2](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_080.jpg)

### Wrap up: hardware supported multithreading

**Interleaved multithreading (IMT)** runs only one thread's one (no superscalar) or multiple (superscalar) instruction(s) in a clock.

If a core can run multiple threads' instructions simultaneously (must be supported by superscalar), it's called **simultaneous multithreading (SMT)**. Its concept is same as IMT, but just divide the time slice from 1 clock to 1/n clock.

SMP can make better use of ILP (superscalar), because the instructions of one thread may rely on previous instructions, meaning that they cannot run in one clock, if the CPU doesn't support SMP, then even though its superscalar can theoretically run 8 instructions per clock, it may run 3 in reality. But it it supports SMP, it can run 3 instructions from thread 1, 3 instructions from thread 2, and 2 instructions from thread 3 (most CPUs only support running 2 threads simultaneously). This makes 100% use of superscalar! For more information about [IMP and SMP](https://gfxcourses.stanford.edu/cs149/fall21/lecture/multicorearch/slide_78) professor Kayvon Fatahalian explained them in detail here.

![hardware supported multithreading](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_081.jpg)

Here is imaged chip, it has 16 core, supporting 8 SIMD ALUs and 4 threads per core. So it can handle maximum 16 \* 8 = 128 pieces of data in parallel. And it's better to give it 128 \* 4 =  512 piece of data to hide latency (moreover, considering that it supports superscalar, it would be even faster):

![a imaged chip](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_082.jpg)

Here is an example of real world core (and a chip have multiple cores). The fetch/decode units here is to find out which instructions can run in parallel to fill the ALUs.

![an real world example: a intel core](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_083.jpg)

## Recall and summary

Watch this part: [![wrap up][yt]](https://youtu.be/F4bVSyz_jxo?t=1718).

- **Superscalar, decided by both the number of execution units (ALUs) and the number of fetch/decode units**. Although each execution unit (ALU) can do only one thing in a clock, we have multiple ALUs, so that a core can do multiple instructions in one clock. And fetch/decode is used to figure out which instructions can run in one clock.
  > In the context of this course, number of fetch/decode units and number of groups of ALUs should be the same, but in complicated real world chips, they are not necessary to be the same. For example, the picture above shows 6 fetch/decode units and 7 groups of ALUs.
- **Multicore, decided by how many cores.**
- **SIMD, decided by how many ALUs are bunched in a group.** SIMD uses only one fetch/decode unit!
- **Multithreading, decided by number of execution contexts.** It can be used to hide stall.

### GPU's SIMT

SIMT: single instruction multiple threads. Many GPUs don't use vector instructions, instead, they use scalar instructions. When multiple thread's PC are same, the GPU can execute them simultaneously with SIMT ALUs.

## Thought experiments

![thought experiments p1](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore/images/slide_103.jpg)
Answer: [![thought experiment p1][yt]](https://youtu.be/F4bVSyz_jxo?t=2103)

Let's start with this thought experiment:

![thought experiments p2](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore2-ispc/images/slide_003.jpg)
My answer is yes, because they can calculate independently (can do parallelization), and they share same instructions and data can be fetched in batch as vector (SIMD), of course, we can also use superscalar, multithreading, multicore on it! \

Answer: yes [![thought experiment p2][yt]](https://youtu.be/F4bVSyz_jxo?t=2310)??? \[[Quick jump](#look-back-to-thought-experiment)\]

## Latency and bandwidth

Here is an example of latency and bandwidth (throughput):
![latency and bandwidth](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore2-ispc/images/slide_008.jpg)

You can improve bandwidth by

- Driving faster (improve bandwidth and reduce latency)
- Building more lane (improve bandwidth, but cannot change latency)
- Using the road more efficiently (improve bandwidth significantly, don't change latency)

![how to improve bandwidth](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore2-ispc/images/slide_009.jpg)
![how to improve bandwidth 2](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore2-ispc/images/slide_010.jpg)

### Laundry example

![laundry example](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore2-ispc/images/slide_013.jpg)

You can improve its throughput by building a pipeline:

![laundry pipeline](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore2-ispc/images/slide_015.jpg)

Although washer is used 100%, washer and student still have some idle time. (-> We can have 4 dryers, 3 washer, and 1 student. So that the through put is 12 clothes per 3 hour (4 clothes / hour)! Nothing / nobody goes idle at any time.)

## Pipeline

***The concept of a pipeline is: don't let anything idle, make 100% use of them.***

### Limitation of a pipeline

If we apply pipeline to computer.

![an computer example](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore2-ispc/images/slide_017.jpg)
![apply pipeline to a computer program](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore2-ispc/images/slide_018.jpg)

As you can see, in this example, the throughput of this program is **limited by the throughput of data fetching**! The following image shows how sever this limitation is.

![throughput of a pipeline is limited by the slowest part](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore2-ispc/images/slide_020.jpg)

### Look back to thought experiment

So, look back to the [though experiment](#thought-experiments), the GPU needs 98TB/sec memory bandwidth to get busy, but the fastest memory human can build is 900GB/sec. So, due to the limitation of memory operation, we only make 1% use of GPU! So, this program is actually a **very bad** program and people seldom realize it.

![back to thought experiment](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore2-ispc/images/slide_022.jpg)

***Bandwidth is usually the biggest problem people are facing.***

Before new memory system comes out, what people can do is changing their program, making it access memory infrequently.

![try to write a program with infrequent memory access](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore2-ispc/images/slide_024.jpg)

## ISPC Programming Abstractions

Students are always confused about abstraction and implementation. Professor Kayvon Fatahalian believes that learn ISPC will make these concepts very clear in students' mind.

### Abstraction vs implementation

![abstraction vs implementation](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore2-ispc/images/slide_027.jpg)

### ISPC

Professor Kayvon Fatahalian: *"You cannot find a lot about ISPC on Reddit or StackOverflow, because its number of users is number of students on CS149 * number of years of CS149 + 100 to 200 who want to get really good performance on an Intel chip."* XD. C++ cannot run as fast as ISPC, there are a lot of reasons why C++ is hard to parallelize.

[![ISPC][yt]](https://youtu.be/F4bVSyz_jxo?t=3927) Before calling ISPC functions, there is only one thread and one control flow, but when ISPC function is called, multiple control flows run in parallel.

![control flow of ISPC](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore2-ispc/images/slide_033.jpg)

Notice that ISPC doesn't use the word "SIMD" or "thread", they use "program instance", "gang", and "task" (a bunch of instances). Because "thread" and "SIMD" are **implementations**, while "program instance", "gang", and "task" are **abstraction**.

| ISPC abstraction | Implementation |
| :---             | :---           |
| Task             | Thread         |
| Gang             | SIMD           |
| Program instance | SIMD lane      |

### Implementation of ISPC can vary -> `foreach()` hides the implementation

e.g. You can created a interleaved assignment or a blocked assignment:

```cpp
// Interleaved:
for (uniform int i = 0; i < N; i += programCount) {
  f(i + programIndex);
}

// Blocked:
uniform int count = N / programCount;
int start = programIndex * count;
for (uniform int i = 0; i < count; ++i) {
  f(start + i);
}
```

![interleaved](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore2-ispc/images/slide_035.jpg)
![blocked](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore2-ispc/images/slide_038.jpg)

If you just don't want to take care of implementation, you just want to work to be done by several program instances. Then `foreach()` comes to help. You just tell ISPC here is a bunch of work to be done by the whole gang, and they are independent. Then, let ISPC finish the implementation.

Here is a `foreach()` code example, ISPC will decide the assignment automatically:
![foreach()](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore2-ispc/images/slide_041.jpg)

All following grey boxes are possible implementations of pink box (foreach), the ISPC compiler may choose any of them. This image is just telling you that implementation can vary, but we just need to care about abstraction.
![possible implementations of foreach()](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore2-ispc/images/slide_042.jpg)

## Side talks

When we talk about "complete a multiply operation per clock", we are saying its throughput, rather than its latency.
![instruction pipeline](https://gfxcourses.stanford.edu/cs149/fall23content/media/multicore2-ispc/images/slide_025.jpg)
[![instruction pipeline][yt]](https://youtu.be/F4bVSyz_jxo?t=3624)

<!----------- References ----------->
[yt]: https://img.shields.io/badge/YouTube-%23FF0000.svg?style=flat-square&logo=YouTube&logoColor=white
