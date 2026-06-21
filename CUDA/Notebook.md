# CUDA Programming Course – High-Performance Computing with GPUs - by Elliot Arledge

[Course Link](https://youtu.be/86FAWCzIe_4)
[Github - CUDA course](https://github.com/Infatoshi/cuda-course)
[Github - MNIST](https://github.com/Infatoshi/mnist-cuda)

## Chapter 00 - Intro

### Prerequisites

- Access to GPU
- Python
- Basic differentiation
- Vector calculus
- Linear algebra

### Takeaways

The main GPU performance bottleneck is memory bandwidth — the communication between chips and inside chips. We will try to optimize it.

Another hard things to do is how to bring a new algorithm to life, like implementing it on PyTorch. We will do it, too.

And we will learn Karpathy's llm.c [![Karpathy's llm.c repository][github]](https://github.com/karpathy/llm.c).

## Chapter 01 - Deep Learning Ecosystem

Note book here: [![notebook][github]](https://github.com/Infatoshi/cuda-course/tree/master/01_Deep_Learning_Ecosystem)

### Research

- **PyTorch** (by Meta): Learn PyTorch at [![Learn PyTorch][yt]](https://youtu.be/Z_ikDlimN6A)
- TensorFlow (by Google)
- Keras: TensorFlow's version of `PyTorch.nn`
- JAX (by Google): used to accelerate linear algebra. JAX is almost identical to NumPy [![JAX in 100 seconds][yt]](https://youtu.be/_0D5lXDjNpw). X: accelerate; A: Autograd, allows you to automatically differentiate functions; J: JIT (Just-in-Time) - compilation. Notice that JAX arrays are immutable.
- MLX (by Apple): for Apple Silicon
- PyTorch Lightning: reduce boilerplate code.

### Production

Training and inference.

- Inference-only
  - vLLM
  - TensorRT (by Nvidia)
- Triton (by OpenAI)
- torch.compile: can increase performance by 30% with just one line
- TorchScript (old)
- ONNX Runtime (by Microsoft)
- Detectron2 (by Meta)

### Low-Level

- **CUDA** (for Nvidia GPUs)
- ROCm (for AMD GPUs)
- OpenCL

### Inference for Edge Computing & Embedded Systems

Edge computing: de-centralized computing.

- CoreML (for Apple devices)
- PyTorch Mobile
- TensorFlow Lite

### Easy to Use

- FastAI
- ONNX (Open Neural Network eXchange)
- wandb (weights and biases)

### Cloud Providers

- **AWS**
- Google Cloud
- Microsoft Azure
- OpenAI
- VastAI
- Lambda Labs

### Compilers

- XLA: for JAX
- LLVM: for C/C++
- MLIR
- **NVCC: Nvidia CUDA Compiler**

### Misc

- [**Huggingface**](https://huggingface.co/)

## Chapter 02 - CUDA Setup

WSL -> python3 -> cuda toolkit and nvidia driver (follow the [Nvidia instructions](https://docs.nvidia.com/cuda/wsl-user-guide/index.html)).

For WSL, only install driver on Windows. Use `nvidia-smi` to check if driver is installed successfully. `CUDA version` is the maximum supported CUDA version.

Use `nvcc --version` to check if CUDA toolkit is installed successfully. This version should be less than or equal to the `CUDA version` shown in `nvidia-smi`.

<!----------- References ----------->
[yt]: https://img.shields.io/badge/YouTube-%23FF0000.svg?style=flat-square&logo=YouTube&logoColor=white
[github]: https://img.shields.io/badge/-dotfiles-white?style=flat&logo=github&logoColor=181717
