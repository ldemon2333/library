The NVIDIA GPU architecture is built around a scalable array of multithreaded Streaming Multiprocessors (SMs). 

A mutiprocessor is designed to execute hundreds of threads concurrently. To manage such a large number of threads, it employs a unique architecture called _SIMT_ (_Single-Instruction, Multiple-Thread_) that is described in [SIMT Architecture](https://docs.nvidia.com/cuda/cuda-c-programming-guide/#simt-architecture).

## 4.1 SIMT Architecture
The multiprocessor creates, manages, schedules, and executes threads in groups of 32 parallel threads called _warps_. Individual threads composing a warp start together at the same program address, but they have their own instruction address counter and register state and are therefore free to branch and execute independently.

A warp executes one common instruction at a time, so full efficiency is realized when all 32 threads of a warp agree on their execution path. If threads of a warp diverge via a data-dependent conditional branch, the warp executes each branch path taken, disabling threads that are not on that path. Branch divergence occurs only within a warp; different warps execute independently regardless of whether they are executing common or disjoint code paths.

SIMT 架构类似于 SIMD（单指令、多数据）向量组织，因为一条指令控制多个处理元素。一个关键的区别是 SIMD 向量组织向软件公开 SIMD 宽度，而 SIMT 指令指定单个线程的执行和分支行为。与 SIMD 向量机相比，SIMT 使程序员能够为独立的标量线程编写线程级并行代码，以及为协调线程编写数据并行代码。为了正确性，程序员基本上可以忽略 SIMT 行为；但是，通过注意代码很少需要 warp 中的线程发散，可以实现显着的性能改进。在实践中，这类似于传统代码中缓存行的作用：在设计正确性时可以安全地忽略缓存行大小，但在设计峰值性能时必须在代码结构中考虑缓存行大小。另一方面，矢量架构要求软件将负载合并到矢量中并手动管理发散。

在 NVIDIA Volta 之前，warp 使用一个程序计数器，该计数器在 warp 中的所有 32 个线程之间共享，并使用活动掩码来指定 warp 的活动线程。因此，来自不同区域或不同执行状态的同一 warp 的线程无法相互发送信号或交换数据，并且需要通过锁或互斥锁保护的细粒度数据共享的算法很容易导致死锁，具体取决于竞争线程来自哪个 warp。

从 NVIDIA Volta 架构开始，独立线程调度允许线程之间完全并发，而不管 warp 如何。使用独立线程调度，GPU 可以维护每个线程的执行状态，包括程序计数器和调用堆栈，并且可以以每个线程的粒度执行，以便更好地利用执行资源或允许一个线程等待另一个线程生成数据。调度优化器确定如何将来自同一 warp 的活动线程分组到 SIMT 单元中。这保留了 SIMT 执行的高吞吐量，就像之前的 NVIDIA GPU 一样，但具有更大的灵活性：线程现在可以在子 warp 粒度上发散和重新收敛。

如果开发人员对以前硬件架构的 Warp 同步性做出假设，则独立线程调度可能会导致参与执行代码的线程集与预期大不相同。特别是，任何 Warp 同步代码（例如无同步、Warp 内缩减）都应重新审视，以确保与 NVIDIA Volta 及其他版本的兼容性。

## 4.2 Hardware Multithreading
