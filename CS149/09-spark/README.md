# Lecture 9 - Distributed Data-Parallel Computing Using Spark

This lecture's [slides](https://gfxcourses.stanford.edu/cs149/fall23/lecture/spark/).

## Today's goal

Learn how to programming on distributed systems:

![goal](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_002.jpg)

## Why use a cluster

It has huge I/O bandwidth:
![why](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_003.jpg)

Notice that I/O is not same as memory. We have talked about computing, memory. But we haven't talked about I/O.

But you also need to program more careful when dealing with multiple machines.

## Warehouse-scale computers

Scientists found that the difference between a Warehouse-scale computers (WSC) and a supercomputer is network. And WSCs spend a lot of money on network.
![WSC](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_005.jpg)

### WSC network

![network](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_006.jpg)
The I/O of a node has two components: network and SSD. Both are not introduced before in this course.

Memory bandwidth (100GB/s) is much greater than I/O bandwidth.

The I/O bandwidth between different racks were 0.1 GB/s in the past. But now, it became 2 GB/s, which is compatible with intra-rack communication.

### Message passing

Because each node is a machine running on a OS, they cannot share memory. We need another communication mechanism called **message passing**.
![message passing](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_007.jpg)

Question: if you have message passing, do you need synchronization?

- My answer: yes. Because if you don't synchronize, when the other machine (thread) is still receiving data. The sender modifies the data. Then the received data is contaminated.
- Professor: you do NOT need explicit synchronization. `send()` and `recv()` themselves are synchronized. They will proceed only after data sending are done.

### Storage system

Nodes and even racks may fail. How to ensure we don't lose data? -> Let's build a distributed file system.
![distributed file system](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_008.jpg)
![distributed file system 2](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_009.jpg)

The idea of distributed files system is that you have a global namespace, and break data into blocks, then distribute the multiple copies of blocks in different machines.

![idea of distributed file system](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_010.jpg)

## MapReduce

### Example: CS149 website

The CS149's website's log file is very large and is distributed at different malines.
![example](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_013.jpg)
Now, professor wants to know more about website users. For example, what phone are students using.
![example](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_014.jpg)

You could use Message Passing Interface (MPI). But "believe me, that's painful. Furthermore, it wouldn't necessarily handle fault tolerance"

Instead, you can use some ideas in [data parallel](../08-dataparallel/README.md) operations. **Map** and **Reduce** (called Fold in last lecture).

### Map

Review:
![Map](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_015.jpg)

Why map is good?

- It is easily parallelizable.
- It has no side effect, meaning it won't change its input. So you can use Map as often as you like.
- It has some fault-tolerant benefits (will be covered later).

### Reduce (Fold)

Review:
![Reduce](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_016.jpg)

### MapReduce example

Here is a MapReduce example:
![MapReduce](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_017.jpg)

Word count:
![word count](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_019.jpg)

- Step 1: run mapper function
- Step 2: gather data, store them in our distributed file system
- Step 3: run reducer. Reducer fetch data from file system.

It is easy to parallelize mappers. But how to parallelize reducers? -> All keys with same values go to same reducer. For example, in the image above, all pairs whose keys are "brown", "fox", "how", "now", "the" goes to reducer 0, all pairs whose keys are "ate", "cow", "mouse", "quick" goes to reducer 1. So, MapReduce is also called "Map-GroupByKey-reduce" or "Map-Sort-Reduce"

### Step 1: run the mapper function

![step 1](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_020.jpg)
Question: how to assign work to get load balance?

- Idea 1: use work queue
- Idea 2: each core process data store on this machine

**Notice that idea 1 need the support of very powerful network. If network is not strong enough, use idea 2.**

### Step 2 and 3: gathering data, running the reducer

![idea 2](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_021.jpg)

Question 1: how to assign tasks to reducers? Use a scheduler to assign tasks to reduce workers.

Question 2: how to make sure all tasks goes to correct reducer? Use a hash function based on key value. For example: send key "Safari iOS" to node 0.
![hash function](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_022.jpg)
Some system may decided which key goes to which node very early. Some may decide after all mappers done their work.

\[Remember\] You cannot do any reduction before all mapping are done.

### Expend to large scale

![large scale](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_023.jpg)

If we have thousands of nodes, we need to be aware that some nodes may fail. (But we assume that we won't lose data, thanks to our fault-tolerant distributed file system. We just may lose some computation unites.)

Moreover, not all nodes in a data center have same computational power. Some newer chips may be faster, and some older chips may be slower.

![scheduler's responsibility](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_024.jpg)

### Summary: MapReduce

So, with the help of MapReduce, we can
![MapReduce benefits](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_025.jpg)
(And it's easier than message passing interface.)

But it has limits:

- What you can do is just create a linear model: a Map followed by a Reduce, then Map, then Reduce... (DryadLINQ can extend what you can do, it has academic impact, but it hasn't been adopted to real-world problems)
- Every iteration needs a Hadoop Distributed File System (HDFS) read followed by a HDFS write, which is inefficient.
- If you need to query in some ad hoc way. For each query, you need to read data and do MapReduce, which is also inefficient.
![MapReduce limitations 1](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_026.jpg)
![MapReduce limitations 2](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_027.jpg)

## Memory bandwidth and I/O bandwidth

As we [seen earlier](#wsc-network), memory bandwidth is much faster than I/O bandwidth.

Thus, although there under most cases (image below), we can fit data entirely in memory. **But**, this programming model (MapReduce) forces us to transfer data back and forth between SSD and memory.
![paper](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_029.jpg)

Question: could we come up with a model that allow us use memory more intensively to avoid relatively slower I/O bandwidth?

Let's think the drawback of storing all data (including intermediate results) on memory: "If you lose power, you screw."

## Spark

But we do have a solution: Spark, a in-memory, fault-tolerant distributed computing.
![spark](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_030.jpg)

![goal](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_031.jpg)

### How to make it fault-tolerant

![fault tolerance](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_032.jpg)

### How does Spark implement a fault-tolerant system

Resilient Distributed Dataset (RDD) is a **read-only, ordered** collection of records. RDDs can only be created from other RDDs or persistent storage (e.g. SSD).

![spark's solution](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_033.jpg)

In this image, `lines` is a RDD created from persistent storage. `mobileViews` and `safariViews` are RDDs created from other RDDs. This sequence of creating RDDs is called **lineage**.

You can also combine the lines above into one line, just like pipeline in Java. Where `filter()`, `map()`, `reduceByKey()` are transformers, the sequence of transformer is called lineage. `collect()` is an action, not a transformation
.
>[!Note] transformations transform a RDD to another RDD. action convert a RDD to non-RDD. See [section RDD transformations and actions](#rdd-transformations-and-actions).

![RDD example](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_034.jpg)

### Persist

Persist means: keep the data in memory. With the help of persist, we can create **forks** in Spark. For example:
![persist](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_035.jpg)

If we don't use `persist`, it may not exist in memory when we want to create a fork. If so, we can only fetch `mobileView` from file system on disk.

### RDD transformations and actions

![RDD transformations and actions](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_036.jpg)

### How to implement RDDs

If we want to store every RDDs, it will occupy huge memory space.
![how to implement RDDs](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_038.jpg)

To solve it, we can learn from 2 previous examples:

- Fusion
  ![example 1: fusion](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_040.jpg)
- Tiling
  ![example 2: tiling](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_041.jpg)

### Fusion

If we have narrow-dependent RDDs, Spark can do fusion automatically for us.

Narrow dependency and wide dependency
![narrow RDDs](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_045.jpg)
![wide RDDs](https://gfxcourses.stanford.edu/cs149/fall23content/media/spark/images/slide_046.jpg)

## Summary

We talked about MapReduce and Spark. MapReduce is fault-tolerant but slow, because it stores data on disks. Spark is both fault-tolerant and fast. Because it stores data on memory, but also has some mechanisms to ensure fault tolerance.

We are running out of time. Next lecture will continue.

<!----------- References ----------->
[yt]: https://img.shields.io/badge/YouTube-%23FF0000.svg?style=flat-square&logo=YouTube&logoColor=white
