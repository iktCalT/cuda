# Lecture 5 - Performance Optimization I: Work Distribution and Scheduling

Slides: [Lecture 4](https://gfxcourses.stanford.edu/cs149/fall23/lecture/progbasics/) and [Lecture 5](https://gfxcourses.stanford.edu/cs149/fall23/lecture/perfopt1/)

## Review

How to use only 1 barrier for last lecture's the [example](../04-progbasics/README.md/#what-does-barrier-do-in-shared-address-space). Use separate copies of `diff` for each iteration. [![solution][yt]](https://youtu.be/mmO2Ri_dJkk?t=146) Its concept is same as `myDiff`, just in different context: update private copy of information, and synchronize them at the end.
> [!Note]
> We only need to store 3 versions of `diff`: previous iteration, current iteration, and next iteration.

![solution](https://gfxcourses.stanford.edu/cs149/fall23content/media/progbasics/images/slide_049.jpg)

## Workload assignment and scheduling

Lecture 5 is about **assignment**—how to figure out which worker does which piece of work.

![today's goal](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_002.jpg)

- Lecture 5: How to synchronizing works and how to make workload balance.
- Lecture 6: How to manage memory efficiently.

## Tip 1: use the simplest solution

Don't apply advanced techniques you heard on the lecture blindly to your program.

*Always use the simplest solution to solve it, then use the simplest method to parallelize it. And measure its performance. And only if the performance is bad, you start to do more sophisticated things.* (Usually, the complex method is slower than a simpler one.)

> [!important]
> Always do the simplest things first.

![tip 1](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_003.jpg)

## Balancing the workload

Here is an example of imbalanced workload.
![imbalanced workload](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_004.jpg)
As we can see in this example, unbalanced workload will make the program slow. To balanced workload, we need to divide the work completely evenly.

### Static assignment

Just like what we did in assignment 1, program 1. Note that the workload of each pixel is indicated by its brightness (because it needs more iteration to exit loop). The interleaved assignment is feasible because we assume that adjacent rows have almost same workload.

What we did in that assignment is static assignment:
![static assignment](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_005.jpg)

When the cost of work is predictable, programmers figure out a good assignment.
![when is static assignment applicable](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_006.jpg)

Static assignment can be very fast, because threads don't need communication or synchronization.

Moreover, if you know the average workload of each thread is almost the same, you can still use static assignment.
![when is static assignment applicable 2](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_007.jpg)

### Semi-static assignment

The assignment changes when the condition changes (causing imbalanced workload). When the workload is balanced again, it stop changing assignment and become static again.

![semi-static assignment](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_009.jpg)

Comparison with static assignment:

- Static assignment: the assignment is known at compile time.
- Dynamic assignment: the assignment changes at runtime, but it keeps "static" if the condition doesn't have abrupt change.

### Dynamic assignment

![dynamic assignment](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_010.jpg)
In this example, the lock ensures that no work is done by 2 threads. And all work should be done in the end.

This example can be abstracted to be a work queue. Each thread asks the queue to pop out a work for it.
![dynamic assignment](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_011.jpg)

Example: assignment 1, program 3. Why it works better if we create more tasks than the number of maximum parallel threads of a CPU?  
-> For example, your CPU has 2 cores and support running 4 threads in parallel. If you only launch 4 tasks, then the workload of each task may be imbalanced. Just as the image in [static assignment](#static-assignment) shown.

On the other hand, if we **divide the whole work into many smaller pieces**, the thread who took lighter work will finish sooner, and it will go to do the next work. **In the end, all thread will do almost same amount of work.**

## Optimize a dynamic assignment

So, to choose whether to use dynamic assignment or not, we need to compare:

1. The **benefit** of achieving balanced workload
2. The **cost** of communication (synchronization)

To optimize a dynamic assignment, we also need to balance these two factors.

Let's look deeper into this dynamic assignment example, is there any potential problem with it?
![dynamic assignment fine granularity](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_013.jpg)

> [!Note]
> How to time the overhead of communication: time the whole program and time the useful part (or communication part) [![how to time overhead][yt]](https://youtu.be/mmO2Ri_dJkk?t=1580)

If the useful part costs 90% of time, then we may think our program is good enough. However, if the useful part spends only 40%, then we will consider how to reduce the communication time.

### Choose task size

One way to reduce commination time is to make it less frequently. Don't dived the work into too small pieces.

As we can see, the granularity in the example above is too small. The program spends a lot of time on synchronization. And it can cause huge negative impact to a large parallel machine (recall Amdahl's law).

![greater granularity](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_014.jpg)

**A greater granularity will make the workload less balanced. But it can reduce the cost of communication.**

![take away](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_015.jpg)

### Smarter task scheduling

Here is an example of bad scheduling:
![a bad scheduling](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_017.jpg)

Here is a good example of scheduling. In this case assigning large tasks first is a good strategy. So, **the more you know about your tasks, the better strategy you can come up with**.
![a good scheduling](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_018.jpg)

### Using distributed queues

Each thread has its work queue (or task queue). If the queue is not empty, it fetches work for its own work queue, which doesn't need synchronization. And **only when its work queue is empty, synchronization happens**: it steal a work from other threads.
![distributed queues](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_019.jpg)

Its idea is same as keep a private `myDiff` and only synchronize it when you have to do it in the end.

Another benefit of distributed queue is that **tasks in a work queue don't need to be independent!**
![tasks in a work queue don't need to be independent](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_020.jpg)

### Summary

![summary](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_021.jpg)

## Scheduling fork-join parallelism

So far, in almost all the program, the parallelism comes from **doing the same thing on different pieces of data**. They all have this pattern:
![data parallelism](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_023.jpg)

And there is another way of parallelism: you don't describe independent things, but instead, **you create workers and let them do tasks. It is called fork-join parallelism**:
![fork-join parallelism](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_024.jpg)

### Example: quick sort

Fork-join parallelism is useful in quick sort, because it uses **divide-and-conquer** algorithms.
![quick sort](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_025.jpg)
Although each thread will only generate 2 parallel threads, which cannot saturate a large parallel machine. But each child thread can also spawn two threads! You will soon maximize the utilization of all threads.

And there is only two changes in your source code to make it parallel, which is pretty easy. Use keyword: `cilk_spawn` and `cilk_sync` (supported by new C/C++ compiler). Please read this slide carefully to understand what Cilk is doing:
![cilk](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_026.jpg)
> [!Important]
> We have not said anything about threads. Although `cilk_spawn` looks like spawning a thread, and `cilk_sync` looks like synchronizing a thread. However, what `cilk_spawn` really do is spawning work, which is different from creating a thread (although it may cause thread spawning).

### Control flow

Here is the control flow of a normal call:
![control flow of normal call](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_027.jpg)

And here is the control flow of a Cilk call (`foo`, `bar`, `fizz`, `main` are independent mutually):
![control flow of a cilk call](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_028.jpg)
> [!Note]
> Please note that the image above tells us that, logically, `foo()`, `bar()` and `main` **can** run asynchronous. But it does not mean that they will definitely run on different thread, and doesn't mean that they will run in parallel in the end.

Cilk can have multiple valid implementations:

- If Cilk compiler just do nothing, its compiler just generate a sequentially program, that a valid implementation.
- Or, each `cilk_spawn` spawns a thread, and `cilk_sync` joins all threads, that also a valid implementation.
- Or, just like ISPC task, create several worker threads and a task pool.

### Cilk version of quick sort

Here is a quick sort with Cilk. Note that there is an implicit `cilk_sync` at the end of each function. So, you don't see the `cilk_sync` in this piece of code.
![quick sort with Cilk](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_030.jpg)

Tips for writing a Cilk program:
![tips for writing a Cilk program](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_031.jpg)

### Implementation of cilk_spawn

Although it's a valid implementation use `pthread_create` (C) or `std::thread` (C++) for each `cilk_spawn`. It's not a good idea, because spawning so many threads has huge overhead.

It's better to create a small thread pool and a huge task pool just like ISPC task:
![Cilk implementation](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_033.jpg)
> Difference with ISPC: for ISPC all threads execute same instruction stream, but for Cilk, threads can execute different instruction streams.

### Strategy

Terms—child and continuation:
![terms](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_034.jpg)

Basic model of Cilk: thread 0 first runs a work and push other work into its work queue (later, we will talk about which work to execute first). If thread 1's work queue is empty, it steals a work from thread 0's work queue.
![model](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_039.jpg)

So, there is an implementation detail: which to execute first?
![which to execute first](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_040.jpg)

- If we choose continuation first, thread 0 will throw all tasks into work queue, then execute continuation:
  ![continuation first](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_041.jpg)
- If we choose child first, thread 0 will run `foo(0)` then throw the continuation (for loop, but `i = 1`) into the work queue. Then thread 1 continue to take `foo(1)`, then throw the for loop into the work queue. So on and so forth...
  ![child first](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_042.jpg)

In child first a bunch of tasks will bounce back and forth between work queues. It is not good, but there is [better way](#child-first) to write a child first strategy.

### Continuation first

Let's consider a recursion example, the image shown the work queue of a continuation first method [![benefits of continuation first][yt]](https://youtu.be/mmO2Ri_dJkk?t=3711):
![benefit of continuation first](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_044.jpg)
As we can see, the work is added into work queue in this order: 101-200 -> 51-100 -> 26-50. The larger tasks are added in to work queue first. So other threads can fetch larger tasks very early. There are many **benefits of stealing larger tasks**:

- As we seen in section [smarter task scheduling](#smarter-task-scheduling), it can make workload more balance.
- If you choose a larger task, you don't have to synchronize for a long time (because it will generate smaller tasks in its work queue and try to finish its own tasks first, which doesn't need communication).
- Thread 0 will always fetch from the bottom of its work queue, those are smaller tasks. If other threads also steal smaller tasks, smaller tasks will finish quickly, and they will wait for a thread decompose a larger thread to start working (and this needs communication).

![impregnation: steal](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_045.jpg)
![implementation](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_048.jpg)

#### QA: which queue to steal from

Theoretically, random selection is the most efficient way. [![QA: which queue to steal from][yt]](https://youtu.be/mmO2Ri_dJkk?t=4073)

### Child first

It is better to write child first in divide-and-conquer way. Same as continuation first, it make sure large tasks are spawned early:
![child first](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_049.jpg)
In Cilk, there is a `cilk_for` keyword, this is implemented in this way.

### Implementation of cilk_sync

Keep a record of a spawn counter and a done counter. Whenever a thread steal a work from a thread related to this block, spawn counter plus 1. Whenever a thread related with this block is empty, done counter plus 1. When this two counter are equal, synchronization finishes. [![cilk_sync][yt]](https://youtu.be/mmO2Ri_dJkk?t=4354)

### Summary of Cilk implementation

![summary Cilk implementation](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_061.jpg)
![summary Cilk](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt1/images/slide_062.jpg)

<!----------- References ----------->
[yt]: https://img.shields.io/badge/YouTube-%23FF0000.svg?style=flat-square&logo=YouTube&logoColor=white
