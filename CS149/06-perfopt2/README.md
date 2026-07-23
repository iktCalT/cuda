# Lecture 6 - Performance Optimization II: Locality, Communication, and Contention

You can find [slides here](https://gfxcourses.stanford.edu/cs149/fall23/lecture/perfopt2/).

This lecture is about communication
![topic](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_002.jpg)

## Drawback of shared address space

So far all programs are assumed to use shared address space model.

Although our assumption that "all threads are sharing shared address space" is **conceptually simple**, its **implementation is really complicated**.
![implementation of address space](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_004.jpg)

Here are some examples of how Intel Sun handle the communication (just to tell you that communication is complicated):
![Intel example](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_006.jpg)
![Sun example](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_007.jpg)

Communication between CPUs is also complicated, here is an motherboard can load 2 CPUs:
![non-uniform motherboard](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_008.jpg)

![summary shared address space](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_009.jpg)

## Message passing

In **shared address space** model, communication is **implicit**, meaning a thread can communicate with other threads by modifying data in shared memory space. The drawback of shared address space, as stated above, is its **complexity**.

We can also use **explicit** communication: send message to other threads directly—**message passing** model.
![message passing](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_010.jpg)

> [!Note]
> Notice that in message passing model, each thread has its own **private** address space. And this *private address space* is relatively easy to implement.

![message passing model](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_011.jpg)

There are multiple ways to implement message passing. You can use TCP/IP or UDP to pass between remote machines, or some method to pass between threads. But we don't care about it for now.

### Example of message passing

Let's look back an example shown in [Lecture 4](../04-progbasics/README.md/#a-parallel-program-example). Let's assume we are solving this problem on two machines, connected with Ethernet:
![message passing example](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_014.jpg)
![message passing example physical model](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_015.jpg)

Each thread need other threads to send extra data to it, so that it can run correctly (the extra space used to hold data from other threads are called "ghost rows", "ghost cells", "ghost values", etc.):
![data replication](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_018.jpg)

Here is how to write a message passing code:
![code for message passing](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_019.jpg)
> Macro: `MSG_ID_*` indicates what kind of message it is.  
> Send: `send(data from where, size of data, to which thread, message type)`. Notice they send `&localA[1, 0]` because 0-th row is reserved for ghost values. First and last threads also reserved 2 ghost rows, although they only need 1 (to keep the code simpler).  
> Receive (wait until received specific type of message from specific thread): `recv(store to where, size of data, from which thread, message type)`.

Compared with the code in *shared address space* model, there is no lock and barriers in this model. So, how to synchronize? In other words, how do we know this loop finishes?  
-> Thread 0 gather diff from other threads, and determine whether to exit. Then, send the message (exit or not) to other threads.

- No lock: because nothing is shared
- No barrier: because synchronize by messaging
- Same indices for all threads (from 0 to n+2). While shared space use different indices (thread 0 from 0 to n-1, thread 1 from n to 2*n-1, ...). Because address space is private.

![notes on message passing](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_020.jpg)

\[Note\]: explain `send` and `recv` (synchronous version, [asynchronous version](#solution-2) in next section) in detail:
![send and receive](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_021.jpg)
> If never received (e.g. network failure), in our simple model, receiver never sends ack, sender and receiver will wait forever. (Of course, we can modify its behavior if this happens).

### Fatal bug in the example code

**Deadlock**: at the beginning of this program, every thread is sending message and waiting for response, no one is receiving. So, **every thread will wait forever!**

#### Solution 1

To fix it, you can pair every two threads and let one send one receive.
![fixed message passing code](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_023.jpg)

#### Solution 2

Or, you can use asynchronous message passing process.
![asynchronous message passing](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_024.jpg)
> [!Note]
> Don't modify `foo` between `SEND returns handle h1` and `Call CHECKSEND(h1)`. Otherwise, you don't know whether the receiver receives the version of `foo` before or after modifying.

## Memory

Remember: message passing is a abstract concept. It can happen under many different context:
![context of message sending](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_025.jpg)
![context of message sending](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_026.jpg)

So, we need to face the memory limit again. No matter in message passing model or in shared space model. Remember that shared space model communicate by reading and writing data implicitly.

Recall:
![memory limit](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_029.jpg)
![Question](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_030.jpg)
Answer: [![question][yt]](https://youtu.be/Mhdny2JNhmc?t=2664). Remember that multithreading can hide stall (high latency), but cannot hide low bandwidth.

![arithmetic intensity](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_031.jpg)

## Two reasons for communication

You may have two reasons for communication:

- inherent: if communication doesn't happen, the program cannot run correctly.
- artificial: all other communication.

### Reduce inherent communication

Left is better than right, and they are both proportional to `N/P`:
![reduce inherent communication](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_034.jpg)

The following is even better than above (proportional to `N/sqrt(P)`), because square has greater area-parameter ratio than rectangular:
![better assignment](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_035.jpg)
> [!Tip]
> *When you are facing bandwidth bound, anything you do to increase arithmetic intensity will be translated to enhanced performance*

### Artifactual communication

![artifactual communication](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_036.jpg)

Artifactual communication can be caused by cache:
![artifactual communication arise from cache](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_038.jpg)
![artifactual communication arise from cache](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_039.jpg)
![artifactual communication arise from cache](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_040.jpg)

Other examples:
![examples of artifactual communication](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_041.jpg)

### Reduce artifactual communication

1. Cache blocking: The most important technique in matrix/tensor operation.
  ![reduce artifactual communication](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_043.jpg)

1. "Fusing" loops: deep learning compilers are starting to support this (they automatically fuse operations for you).
  In this example, instead of calling `add`, `mul` one by one, fusing them together is a better way. Because so that they can share the same loop. Fewer data access operations but same amount of arithmetic operations -> greater arithmetic intensity.
  ![inefficient](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_044.jpg)

## Contention

Read slides. [Starting from here](https://gfxcourses.stanford.edu/cs149/fall23/lecture/perfopt2/slide_46).

For example: office hour
![contention example](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_047.jpg)

Without appointment:
![contention without appointment](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_048.jpg)

With appointment:
![contention without appointment](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_049.jpg)

So, we need to avoid too many requests to a resource at the same time (that's why [randomly steal work from work queue is theoretically faster](../05-perfopt1/README.md/#qa-which-queue-to-steal-from)).
![contention](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_051.jpg)
![contention](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_050.jpg)

## Summary

![summary](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_052.jpg)

## Tips

Always try simple things first:
![tip 1](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_054.jpg)

How to know if I meet bandwidth bound (watch the video [![tip 2][yt]](https://youtu.be/Mhdny2JNhmc?t=4113)):
![tip 2 p1](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_055.jpg)

Left side is facing bandwidth bound: when you increase arithmetic intensity (operational intensity), you are asking CPU to do more calculation with same memory request. So, your program can run faster.  
Right side is facing compute bound: your CPU is 100% used. Even if you ask it to do more work with same amount of memory request, it cannot perform better.
![tip 2 p2](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_056.jpg)
> [!Tip]
> When you are on the left side (rising part) of this graph, you are facing **bandwidth bound**, and you can use techniques like "fusing" loops to increase arithmetic intensity (same throughput, but more computation) and get better performance.  
> However, when you are on the right side (platform) of this graph, you are facing compute limit (your CPU cannot compute faster). Then increasing arithmetic intensity cannot help.
>
> In any case, you can use better algorithm to make it possible to get same result with fewer calculations. But arithmetic intensity may be greater or lesser, depends on your algorithm. It means that in this graph, FLOPS (y axis) may increase or decrease. However, since your algorithm make it possible to finish more work with same FLOPS, you may observe better performance in the end! So, keep in mind that **efficiency is important**!

### How to know which part is bottleneck

Computation, memory bandwidth, locality, synchronization... all these factors can be bottleneck of you program, you can do the following steps to determine which one is affects most.
![get higher performance tips](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_058.jpg)

Profilers are also helpful.
![profilers](https://gfxcourses.stanford.edu/cs149/fall23content/media/perfopt2/images/slide_059.jpg)

## QA

- Two questions about why we use message passing and one question about performance: [![QA][yt]](https://youtu.be/Mhdny2JNhmc?t=2361)

## Bonus slides

Read slides 60 to 69 on [course website](https://gfxcourses.stanford.edu/cs149/fall23/lecture/perfopt2/).

<!----------- References ----------->
[yt]: https://img.shields.io/badge/YouTube-%23FF0000.svg?style=flat-square&logo=YouTube&logoColor=white
