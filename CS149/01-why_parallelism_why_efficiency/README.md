# Lecture 1 - Why Parallelism? Why Efficiency?

A parallel computer is a collection of processing elements that cooperate to solve problems quickly.
![parallel computer](../static/01_parallel-computer.png)

## Demos

Demo 1: 1 person add up 16 numbers spent 45 seconds, while 2 people spent 41 seconds, in which calculation time was only 23 seconds, and communication took 18 seconds. Constraint: **communication**.

Demo 2: one person as assigned with a lot of numbers, and those numbers are large, hard to calculate, which makes his calculation slow. Constraint: **unbalanced workload**.

There are many strategies to distribute works.

Demo3: massive parallel execution. Stanford students are so creative. [![demo 3][yt]](https://youtu.be/V1tINV2-9p4?t=1276) The **dependency** that gets everybody involved makes this creative algorithm actually slow. In other algorithms, **communication** is a severe limitation.

*Although this class is about parallelism, the reality is that it's really about communication and moving things around.*

## Course themes

![course theme 1](../static/01_course-theme-1.jpg)
![course theme 2](../static/01_course-theme-2.jpg)
![course theme 3](../static/01_course-theme-3.jpg)

## Course arrangement

[![course arrangement][yt]](https://youtu.be/V1tINV2-9p4?t=2078)

## Why parallelism?

In the past, single threaded CPU performance increase so rapidly that working on parallelism is thought not worthy.

But in the recent 15 years, both exploiting instruction-level **parallelism** and increasing CPU clock frequency are the main reason for processor performance improvement.

### What is a computer program?

It's a sequence of instructions.

### What a processor do?

Execute instructions.

### What does executing instructions mean?

After some operations, resulting the changes in state (the value of a register or a value in memory).

![processor](../static/01_processor.jpg)

### Putting them together

*A computer program is a list of instructions to execute.*

*An instruction describes an operation for a processor to perform. Executing an instruction typically modifies the computer's state.*

*A state means the values of a program data, which are stored in memory or processor's register.*

### If we have multiple processors

If we have 2 processors, we can turn a 5 clock cycle program into 3 clock cycle. Because instruction 1, 2, 3 are independent.

![two processors](../static/01_two-processors.jpg)

But we cannot improve the performance if we have 3 processors.

![three processors](../static/01_three-processors.jpg)

Because of their dependency. Instruction 4 and 5 are dependent to each other.

![dependency](../static/01_dependency.jpg)

### Idea 1 - Superscalar

So we get our idea 1: there are some hardware (e.g. Intel Pentium CPU) can automatically find independent instructions and execute multiple independent instructions in one clock cycle. It is finished automatically by processor. Programmers don't need to explicitly write parallel program (and don't even feel it). [![idea 1][yt]](https://youtu.be/V1tINV2-9p4?t=3259)

![idea 1](../static/01_idea-1.jpg)

Scientists found that for most ILP (instruction level parallelism), once you can speed it up around 3 times, it will be hard to increase. Even though you increase your CPU's superscalar. Because they don't have more parallelism to be used, just like the 3 processor case above. [![limits of superscalar][yt]](https://youtu.be/V1tINV2-9p4?t=3340)

![recent years](../static/01_recent-years.jpg)

In recent years, due to the "power wall", processors' frequency is no longer growing as fast as before. **So, in recent years, the only way to increase performance is increasing programs' parallelism.**

![why parallelism](../static/01_why-parallelism.jpg)

## What you will learn in this class

### How to write parallel code

You will find out that if you write your C++ (not Python or Lua) code properly, you can speed your program up 32 - 40 times even if you only have 4 cores.

### Efficiency

*But in modern computing software must be more than just parallel... **IT MUST ALSO BE EFFICIENT**.*

*Achieving efficient processing almost always comes down to accessing data efficiency* -> moving data to the right place is going to be the most important thing in this class.

## Memory and cache

### What is memory?

DRAMs and caches (SRAMs) are implementations of memory.

In a programmer's perspective, memory is a sequence of values. And values have two places to be stored in — registers or memory.

DRAMs are slow, and caches are fast

[*Class finishes]

For more information, read CSAPP Chapter 6: The Memory Hierarchy.

<!----------- References ----------->
[yt]: https://img.shields.io/badge/YouTube-%23FF0000.svg?style=flat-square&logo=YouTube&logoColor=white
