 将更多的晶体管投入到数据处理中，例如浮点计算，有利于高度并行计算；GPU 可以通过计算隐藏内存访问延迟，而不是依赖大数据缓存和复杂的流控制来避免长内存访问延迟，这两者都需要耗费大量的晶体管。
![[Pasted image 20250319162137.png]]

# CUDA
a general-Purpose Parallel computing platform and programming Model.

The challenge is to develop application software that transparently scales its parallelism to leverage the increasing number of processor cores.

The Cuda parallel programming model is designed to overcome this challenge.

At its core are three key abstractions  - a hierarchy of thread groups, shared memories, and barrier synchronization - that are simply exposed to the programmer as a minimal set of language extensions.

这些抽象提供了细粒度数据并行和线程并行，嵌套在粗粒度数据并行和任务并行中。它们指导程序员将问题划分为可以通过线程块并行独立解决的粗略子问题，并将每个子问题划分为可以通过块内的所有线程并行协作解决的细小部分。

这种分解允许线程在解决每个子问题时进行协作，从而保留了语言表达能力，同时实现了自动可扩展性。实际上，每个线程块都可以在 GPU 内任何可用的多处理器上以任何顺序（并发或顺序）进行调度，因此编译后的 CUDA 程序可以在任意数量的多处理器上执行（如图 3 所示），并且只有运行时系统需要知道物理多核处理器数量。
![[Pasted image 20250319163314.png]]
这种可扩展的编程模型允许 GPU 架构通过简单地扩展多处理器和内存分区的数量来覆盖广泛的市场范围：从高性能发烧友级 GeForce GPU 和专业 Quadro 和 Tesla 计算产品到各种廉价的主流 GeForce GPU（请参阅支持 CUDA 的 GPU 以获取所有支持 CUDA 的 GPU 的列表）。

# 2 Programming Model
## 2.1 Kernels

## 2.2 Thread Hierarchy

There is a limit to the number of threads per block, since all threads of a block are expected to reside on the same streaming multiprocessor core and must share the limited memory resources of that core. On current GPUs, a thread block may contain up to 1024 threads.

However, a kernel can be executed by multiple equally-shaped thread blocks, so that the total number of threads is equal to the number of threads per block times the number of blocks.

Blocks are organized into a one-dimensional, two-dimensional, or three-dimensional grid of thread blocks. 块被组织成一维、二维或三维的线程块网格，如图 4 所示。网格中的线程块数量通常由正在处理的数据的大小决定，该数量通常超过系统中的处理器数量。
![[Pasted image 20250320140730.png]]


The number of threads per block and the number of blocks per grid specified in the `<<<...>>>` syntax can be of type `int` or `dim3`. Two-dimensional blocks or grids can be specified as in the example above.

Threads within a block can cooperate by sharing data through some _shared memory_ and by synchronizing their execution to coordinate memory accesses. More precisely, one can specify synchronization points in the kernel by calling the `__syncthreads()` intrinsic function; `__syncthreads()` acts as a barrier at which all threads in the block must wait before any is allowed to proceed. [Shared Memory](https://docs.nvidia.com/cuda/cuda-c-programming-guide/#shared-memory) gives an example of using shared memory. In addition to `__syncthreads()`, the [Cooperative Groups API](https://docs.nvidia.com/cuda/cuda-c-programming-guide/#cooperative-groups) provides a rich set of thread-synchronization primitives.

the CUDA programming model introduces an optional level of hierarchy called Thread Block Clusters that are made up of thread blocks. Thread blocks in a cluster are also guaranteed to be co-scheduled on a GPU Processing Cluster (GPC) in the GPU.

And a maximum of 8 thread blocks in a cluster is supported as a portable cluster size in CUDA.
![[Pasted image 20250320140747.png]]


## 2.3 Memory Hierarchy
CUDA threads may access data from multiple memory spaces during their execution. Each thread has private local memory. Each thread block has shared memory visible to all threads of the block and with the same lifetime as the block. Thread blocks in a thread block cluster can perform read, write, and atomics operations on each other's shared memory. All threads have access to the same global memory.
![[Pasted image 20250320134954.png]]
There are also two additional read-only memory spaces accessible by all threads: the constant and texture memory spaces. The global, constant, and texture memory spaces are persistent across kernel launches by the same application.


## 2.4 Heterogeneous Programming
Unified Memory provides managed memory to bridge the host and device memory spaces.

## 2.5 Asynchronous SIMT Programming Model
Starting with devices based on the NVIDIA Ampere GPU architecture, the CUDA programming model provides acceleration to memory operations via the asynchronous programming model.

The asynchronous programming model defines the behavior of [Asynchronous Barrier](https://docs.nvidia.com/cuda/cuda-c-programming-guide/#aw-barrier) for synchronization between CUDA threads. The model also explains and defines how [cuda::memcpy_async](https://docs.nvidia.com/cuda/cuda-c-programming-guide/#asynchronous-data-copies) can be used to move data asynchronously from global memory while computing in the GPU.

### 2.5.1 Asynchronous Operations
异步操作定义为由 CUDA 线程发起并像由另一个线程异步执行的操作。在结构良好的程序中，一个或多个 CUDA 线程与异步操作同步。发起异步操作的 CUDA 线程不需要位于同步线程中。

## 2.6 Compute Capability
设备的计算能力由版本号表示，有时也称为“SM 版本”。此版本号标识 GPU 硬件支持的功能，并在运行时由应用程序使用来确定当前 GPU 上可用的硬件功能和/或指令。

The compute capability comprises a major revision number _X_ and a minor revision number _Y_ and is denoted by _X.Y_.

Devices with the same major revision number are of the same core architecture. The major revision number is 9 for devices based on the _NVIDIA Hopper GPU_ architecture, 8 for devices based on the _NVIDIA Ampere GPU_ architecture, 7 for devices based on the _Volta_ architecture, 6 for devices based on the _Pascal_ architecture, 5 for devices based on the _Maxwell_ architecture, and 3 for devices based on the _Kepler_ architecture.

The minor revision number corresponds to an incremental improvement to the core architecture, possibly including new features.

_Turing_ is the architecture for devices of compute capability 7.5, and is an incremental update based on the _Volta_ architecture.

# 3 Programming Interface
The runtime is introduced in [CUDA Runtime](https://docs.nvidia.com/cuda/cuda-c-programming-guide/#cuda-c-runtime). It provides C and C++ functions that execute on the host to allocate and deallocate device memory, transfer data between host memory and device memory, manage systems with multiple devices, etc. A complete description of the runtime can be found in the CUDA reference manual.

## 3.1 Compilation with NVCC
Kernels can be written using the CUDA instruction set architecture, called _PTX_, which is described in the PTX reference manual. It is however usually more effective to use a high-level programming language such as C++. In both cases, kernels must be compiled into binary code by `nvcc` to execute on the device.

### 3.1.1 Compilation Workflow
#### 3.1.1.1 Offline Compilation
Source files compiled with `nvcc` can include a mix of host code (i.e., code that executes on the host) and device code (i.e., code that executes on the device). `nvcc`’s basic workflow consists in separating device code from host code and then:

- compiling the device code into an assembly form (_PTX_ code) and/or binary form (_cubin_ object),
- and modifying the host code by replacing the `<<<...>>>` syntax introduced in [Kernels](https://docs.nvidia.com/cuda/cuda-c-programming-guide/#kernels) (and described in more details in [Execution Configuration](https://docs.nvidia.com/cuda/cuda-c-programming-guide/#execution-configuration)) by the necessary CUDA runtime function calls to load and launch each compiled kernel from the _PTX_ code and/or _cubin_ object.

The modified host code is output either as C++ code that is left to be compiled using another tool or as object code directly by letting `nvcc` invoke the host compiler during the last compilation stage.

Applications can then:

- Either link to the compiled host code (this is the most common case),
- Or ignore the modified host code (if any) and use the CUDA driver API (see [Driver API](https://docs.nvidia.com/cuda/cuda-c-programming-guide/#driver-api)) to load and execute the _PTX_ code or _cubin_ object.