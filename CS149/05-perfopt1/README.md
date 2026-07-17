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

<!----------- References ----------->
[yt]: https://img.shields.io/badge/YouTube-%23FF0000.svg?style=flat-square&logo=YouTube&logoColor=white
