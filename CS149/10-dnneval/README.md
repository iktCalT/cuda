# Lecture 10 - Efficiently Evaluating DNNs on GPUs

This [lecture's slides](https://gfxcourses.stanford.edu/cs149/fall23/lecture/dnneval/).

## Assignment 3 introduction

[![assignment 3][yt]](https://youtu.be/qbKtU0X6-WU?t=34)

## Deep neural network

Mini-intro to convolutional neural networks: [![start][yt]](https://youtu.be/qbKtU0X6-WU?t=360)

Let's consider this graph
![what DNN does](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_004.jpg)

What does DNN do:
![intro to DNN](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_005.jpg)
![intro to DNN](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_006.jpg)
> [!Note]
> The image above shows and a DNN with **convolutional layer** (right).
> A convolutional layer is layer that can apply convolutional operations. For example, in the image, the right one has a layer where every consecutive 3 nodes are connected to the same node of next layer.

Fully connected layers are just matrix product:
![intro to DNN](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_007.jpg)

### Convolution (kernel)

You can also think of it in another way (the image below is an example of **2D convolution**, its effect is blurry):
![example of convolution](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_010.jpg)

By changing the weights, you can have different effect to input. For example, the image below shows kernels for detecting vertical and horizontal gradients:
![another example of convolution](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_011.jpg)

*But in ML, weights are not given by people, they are learned*.
![multilayer convolution](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_012.jpg)

Tensor can be considered to be many filters (convolutional kernel) stacking together:
![multi-dimensional tensor](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_013.jpg)

And we can also stack tensors -> multilayer neural network (DNN):
![DNN](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_014.jpg)
![DNN](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_015.jpg)

Takeaway: *it's important to perform a bunch of convolutions on a input image and producing a bunch of output images, then we need to perform convolutions to them, and so on...*

## Efficiently implementation convolution layers

**Our goal is to make this process faster.**

There are several ways to make it faster:

### Approach 1: use a better network design

![approach 1](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_017.jpg)

Take ML classes and design better networks. For example, ResNet can skip several steps:
![ResNet](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_018.jpg)
People spend years and try many combinations to find out a good network.

Algorithm can bring huge improvement (25x in just 3 years. Hardware cannot grow that fast)
![effective networks over years](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_020.jpg)
![improvement](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_021.jpg)

So, if you are a ML system engineer, you should keep track of the newest algorithms.

### Approach 2: code optimization - highly relevant to this class

![approach 2](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_022.jpg)

In the following example, INPUT_DEPTH is input channels (RGBs). So, it's an 2D convolution on 3D input (2D image + 1D channel).
![example](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_023.jpg)

However, the code above is not efficient. But if you can regard convolution as matrix multiplication. Then you can apply libraries like MatLab, Numpy, PyTorch to make it really fast.

Here is why (with appropriate padding and data duplication) convolution can be implemented as matrix-vector product:
![convolution is matrix multiplication](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_024.jpg)

And multiple convolutions is matrix-matrix product:
![convolution is matrix multiplication](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_025.jpg)

Same as multi-channel case:
![convolution is matrix multiplication](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_026.jpg)

Here is an example: NVIDIA convolution function (GEMM: general matrix multiply). It shows that convolution (y = CONV(x, w)) is equivalent to GEMM (C = GEMM(A, B)):
![NV convolution](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_027.jpg)

## Brief introduction to how to implement these crazy libraries

Do watch the video! [![fast matrix multiplication][yt]](https://youtu.be/qbKtU0X6-WU?t=2044)

### Block

![fast matrix multiplication](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_029.jpg)

If A, B, and C are N*N matrices, how many data will you touch? -> 3 \* N^2. How many operation will you perform? -> N^3.

So, its arithmetic intensity should be O(N). However, the algorithm we wrote show only O(1) arithmetic intensity (hitting bandwidth bound). If you implement them in cache, then arithmetic intensity should be O(N)!

The problem of placing them in cache is just because it's huge matrix. So, we need to partition the the matrix into small sub-matrices so that they can fit in cache!
![partition](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_030.jpg)
Let's assume each block is B\*B size. So, the data we need to touch in each block is 3 \* B^2, and the arithmetic operation is B^3. Thus, the arithmetic intensity is: O(B). **The greater the block size, the greater arithmetic intensity we can achieve.** So, you need to know your cache size, then assign the block size that your cache can just hold blocks A, B, and C. So, the greater the cache size, you greater arithmetic intensity you could achieve. You can achieve 1000x speedup just to make your block size several MB (for example, if each element is int, then you only need 3 * 1000 * 1000 * 4 bytes ≅ 12 MB).

Moreover, to make use of L1, L2, L3 caches, you can create even smaller blocks. And this code can add layers of blocks quickly:
![hierarchical blocked matrix multiplication](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_031.jpg)

### SIMD

Blocking is the easy part, but this is the hard part.

![SIMD](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_032.jpg)
Notice that in this image, A, B, and C are blocks. Not the whole matrices.

You can pick up a vector (= SIMD wide) of B, and a value of A. Then get multiple elements of C with SIMD.

But the problem here is that:

1. We are still go through B in column major order
2. Every instruction here is dependent with previous one, which may hurt ILP (superscalar).

What we can do is to pre-transpose B:
![SIMD 2](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_033.jpg)
Or pre-transpose A and C:
![SIMD 3](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_034.jpg)

Consider these matrix sizes, you need a specific implementation for each combination of matrix sizes. Otherwise, your method may not be optimal.
![matrix sizes](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_035.jpg)

### Memory depletion

The second problem with it is that you duplicate data many times. It will occupy large extra memory. And if you are using ML algorithms like back-propagation, you will run out of memory very quickly.

One solution to this is **thinking it as matrix multiplication, but fetch data from original sources**. It is called **implicit matrix multiply**.
![solution to data duplication](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_036.jpg)
![solution to data duplication](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_037.jpg)
These `.at()` will convert matrix coordinates to real location in memory space. It prevent duplication with the cost of intensive calculation. **To make it faster, you can create a lookup table**.

In this case, you need to make sure the original data are cached. [![cache][yt]](https://youtu.be/qbKtU0X6-WU?t=3036)

### CUTLASS

CUTLASS is a powerful library that can make your matrix multiplication very fast. (But it's not a user-friendly library)
![CUTLASS](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_038.jpg)

### On GPUs

Remember that on GPUs, you have very large cache size:
![GPU cache size](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_039.jpg)
So, you need a lot of work to make use of this large cache (TFLOPS: Tera FLoating-point Operations Per Second):
![fill GPUs with large amount of work](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_040.jpg)

### Wrap up

Although companies like Google, OpenAI are working on optimizing blocking automatically. But for those inner loops, they are best optimized by human.

### Algorithm improvements

For example, with Winograd filter, multiplication can be converted from 6 multiplications + 4 additions to 4 multiplications + 8 additions. Less multiplications but more additions. Whether it will improve performance depends on the machine (FTT, fast fourier transform uses similar method). [![algorithm][yt]](https://youtu.be/qbKtU0X6-WU?t=3370)
![algorithm improvement](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_042.jpg)

### Examples

Libraries like PyTorch, TensorFlow, compile code into lower level DNN specific libraries (provided by hardware vendor), like NVIDIA cuDNN or Intel Deep Neural Network Library.

![cuDNN](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_046.jpg)

- The default (the first one on the list) is "implicit gemm". It means treating the tensor as matrices, but don't really convert them into matrices. -> So that it can use highly efficient matrix libraries, and also save spaces.
- The fourth one, "direct", means use blocking.
- The third one, "gemm", means create real matrices and do matrix-matrix multiplication.

## Memory traffic

We talked about matrix-matrix multiplication, which can be applied to convolutional layers and fully connected layers. But we haven't talked about the fact that there are hundreds of layers back to back. So, when one layer is calculated, then we put it into memory. To calculate next layer, we fetch it back to cache and calculate. This tensor is go in and out of memory over and over again. Moreover, although we can accelerate convolution (first step in the image) by blocking, we cannot do it to scale/bias and max pool (second and third steps).
![memory traffic](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_047.jpg)

What we want to do is do these 3 steps in one step (shown at the bottom of image above).
![fusion](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_048.jpg)

People are trying to make this fusion automatically, but still haven't gotten a good result.

### Attention is all you need

But here is a huge improvement invented in Stanford: "attention".

This is a huge topic (and the slides on website are incomplete), do watch the video! [![attention is all you need][yt]](https://youtu.be/qbKtU0X6-WU?t=3840)
![attention is all you need](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_049.jpg)

Originally, `softmax()` cannot be blocked, because it needs to know the maximum of every row and sum of all `f(x)` (watch video). Which means that you cannot fuse several steps and do many steps one time for a block. You have to calculate the whole layer (blocks are in and out from memory), then do the next step.

However, with attention, you can block `softmax()` (watch the video [![attention is all you need][yt]](https://youtu.be/qbKtU0X6-WU?t=3840) to know how people make it possible). This means that you can store a block in cache, and calculate many steps (fusion), then place it back to memory. And do the same thing to next block.

## Approximation

Approximation is also a method.
![approximation](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_059.jpg)

## Summary

First of all, use good algorithms. If you don't catch up with newest algorithms, you will be left behind. Then, do CS 149 things to optimize computation.
![summary](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_060.jpg)

Note that hardware is also very important [![hardware][yt]](https://youtu.be/qbKtU0X6-WU?t=4511). GPUs have many benefits and many drawbacks for ML.

NVIDIA A100 GPU has tensor cores (NVIDIA's reaction to TPUs). They are specialized cores for ML, which has 8x4 x 4x8 matrix mul-add instructions, making calculation consume much less energy.
![tensor cores](https://gfxcourses.stanford.edu/cs149/fall23content/media/dnneval/images/slide_067.jpg)

<!----------- References ----------->
[yt]: https://img.shields.io/badge/YouTube-%23FF0000.svg?style=flat-square&logo=YouTube&logoColor=white
