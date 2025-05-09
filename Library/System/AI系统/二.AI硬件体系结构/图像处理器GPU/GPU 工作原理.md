
# GPU 工作原理
## CPU vs GPU
现在探讨一下 CPU 和 GPU 在架构方面的主要区别，CPU 即中央处理单元（Central Processing Unit），负责处理操作系统和应用程序运行所需的各类计算任务，需要很强的通用性来处理各种不同的数据类型，同时逻辑判断又会引入大量的分支跳转和中断的处理，使得 CPU 的内部结构异常复杂。

 GPU 即图形处理单元（Graphics Processing Unit），可以更高效地处理并行运行时复杂的数学运算，最初用于处理游戏和动画中的图形渲染任务，现在的用途已远超于此。两者具有相似的内部组件，包括核心、内存和控制单元。

![[Pasted image 20241011115346.png]]

GPU 和 CPU 在架构方面的主要区别包括以下几点：

1. **并行处理能力**：CPU 拥有少量的强大计算单元（ALU），更适合处理顺序执行的任务，可以在很少的时钟周期内完成算术运算，时钟周期的频率很高，复杂的控制逻辑单元（Control）可以在程序有多个分支的情况下提供分支预测能力，因此 CPU 擅长逻辑控制和串行计算，流水线技术通过多个部件并行工作来缩短程序执行时间。GPU 控制单元可以把多个访问合并，采用了数量众多的计算单元（ALU）和线程（Thread），大量的 ALU 可以实现非常大的计算吞吐量，超配的线程可以很好地平衡内存延时问题，因此可以同时处理多个任务，专注于大规模高度并行的计算任务。

2. **内存架构**：CPU 被缓存 Cache 占据了大量空间，大量缓存可以保存之后可能需要访问的数据，可以降低延时。GPU 缓存很少且为线程（Thread）服务，如果很多线程需要访问一个相同的数据，缓存会合并这些访问之后再去访问 DRMA，获取数据之后由 Cache 分发到数据对应的线程。GPU 更多的寄存器可以支持大量 Thread。

3. **指令集**：CPU 的指令集更加通用，适合执行各种类型的任务。GPU 的指令集主要用于图形处理和通用计算。CPU 可以在不同的指令集之间快速切换，而 GPU 只是获取大量相同的指令并高速进行推送。

4. **功耗和散热**：CPU 的功耗相对较低，散热要求也相对较低。由于 GPU 的高度并行特性，其功耗通常较高，需要更好的散热系统来保持稳定运行。

因此，CPU 更适合处理顺序执行的任务，如操作系统、数据分析等；而 GPU 适合处理需要大规模并行计算的任务，如图形处理、深度学习等。在异构系统中，GPU 和 CPU 经常会结合使用，以发挥各自的优势。

 GPU 起初用于处理图形图像和视频编解码相关的工作。GPU 跟 CPU 最大的不同点在于，GPU 的设计目标是最大化吞吐量（Throughput），相比执行单个任务的快慢，更关心多个任务的并行度（Parallelism），即同时可以执行多少任务；CPU 则更关心延迟（Latency）和并发（Concurrency）。

 CPU 优化的目标是尽可能快地在尽可能低的延迟下执行完成任务，同时保持在任务之间具体快速切换的能力。它的本质是以序列化的方式处理任务。GPU 的优化则全部都是用于增大吞吐量的，它允许一次将尽可能多的任务推送到 GPU 内部，然后 GPU 通过大数量的 Core 并行处理任务。

## 并发与并行
1. 并行（Parallelism）

并行指的是同时执行多个任务或操作，通常是在多个处理单元上同时进行。在计算机系统中，这些处理单元可以是多核处理器、多线程、分布式系统等。并行计算可以显著提高系统的性能和效率，特别是在需要处理大量数据或复杂计算的情况下。例如，一个计算机程序可以同时在多个处理器核心上运行，加快整体计算速度。

2. 并发（Concurrency）

并发指的是系统能够同时处理多个任务或操作，但不一定是同时执行。在并发系统中，任务之间可能会交替执行，通过时间片轮转或事件驱动等方式来实现。并发通常用于提高系统的响应能力和资源利用率，特别是在需要处理大量短时间任务的情况下。例如，一个 Web 服务器可以同时处理多个客户端请求，通过并发处理来提高系统的吞吐量。

因此并行和并发的主要区别如下：

- 并行是指同时执行多个任务，强调同时性和并行处理能力，常用于提高计算性能和效率。

- 并发是指系统能够同时处理多个任务，强调任务之间的交替执行和资源共享，常用于提高系统的响应能力和资源利用率。

在实际应用中，并行和并发通常结合使用，根据具体需求和系统特点来选择合适的技术和策略。在实际硬件工作的过程当中，更倾向于利用多线程对循环展开来提高整体硬件的利用率，这就是 GPU 的最主要的原理。

以三款芯片为例，对比在硬件限制的情况下，一般能够执行多少个线程，对比结果增加了线程的请求（Threads required）、线程的可用数（Threads available）和线程的比例（Thread Ration），主要对比到底需要多少线程才能够解决内存时延的问题。从表中可以看到几个关键的数据：

- GPU（英伟达 A100）的时延比 CPU （AMD Rome 7742，Intel Xeon 8280）高出好几个倍数；

- GPU 的线程数是 CPU 的二三十倍；

- GPU 的可用线程数量是 CPU 的一百多倍。计算得出线程的比例，GPU 是 5.6，CPU 是 1.2~1.3，这也是 GPU 最重要的一个设计点，它拥有非常多的线程为大规模任务并行去设计。

|                        | AMD Rome 7742 | Intel Xeon 8280 | NVIDIA A100 |
| ---- | - | --- | -- |
| Memory B/W(GB/sec)     | 204           | 143             | 1555        |
| DRAM Latency(ns)       | 122           | 89              | 404         |
| Peak bytes per latency | 24,888        | 12,727          | 628,220     |
| Memory Efficiency      | 0.064%        | 0.13%           | 0.0025%     |
| Threads required       | 1,556         | 729             | 39,264      |
| Threads available      | 2048          | 896             | 221,184     |
| Thread Ration          | 1.3X          | 1.2X            | 5.6X        |

CPU 和 GPU 的典型架构对比可知 GPU 可以比作一个大型的吞吐器，一部分线程用于等待数据，一部分线程等待被激活去计算，有一部分线程正在计算的过程中。GPU 的硬件设计工程师将所有的硬件资源都投入到增加更多的线程，而不是想办法减少数据搬运的延迟，指令执行的延迟。

相对应的可以把 CPU 比喻成一台延迟机，主要工作是为了在一个线程里完成所有的工作，因为希望能够使用足够的线程去解决延迟的问题，所以 CPU 的硬件设计者或者硬件设计架构师就会把所有的资源和重心都投入到减少延迟上面，因此 CPU 的线程比只有一点多倍，这也是 SIMD（Single Instruction, Multiple Data）和 SIMT（Single Instruction, Multiple Threads）架构之间最大的区别。CPU 不是通过增加线程来去解决问题，增加线程是为了尽可能减少时延，但主要途径还是去优化线程的执行速率和效率，这就是 CPU 跟 GPU 之间最大的区别，也是它们的本质区别。

> SIMD (Single Instruction, Multiple Data) 和 SIMT (Single Instruction, Multiple Threads)
> 
> SIMD 架构是指在同一时间内对多个数据执行相同的操作，适用于向量化运算。例如，对于一个包含多个元素的数组，SIMD 架构可以同时对所有元素执行相同的操作，从而提高计算效率。常见的 SIMD 架构包括 SSE (Streaming SIMD Extensions) 和 AVX (Advanced Vector Extensions)。
> 
> SIMT 架构是指在同一时间内执行多个线程，每个线程可以执行不同的指令，但是这些线程通常会执行相同的程序。这种架构通常用于 GPU (Graphics Processing Unit) 中的并行计算。CUDA (Compute Unified Device Architecture) 和 OpenCL 都是支持 SIMT 架构的编程模型。
> 
> SIMD 适用于数据并行计算，而 SIMT 适用于任务并行计算。

## GPU 工作原理

### 基本工作原理

首先通过 $AX+Y$ 这个加法运算的示例了解 GPU 的工作原理，$AX+Y$ 的示例代码如下：

```c
void demo(double alpha, double *x, double *y)
{
    int n = 2000;
    for (int i = 0; i < n; ++i)
    {
        y[i] = alpha * x[i] + y[i];
    }
}
```

示例代码中包含 2 FLOPS 操作，分别是乘法（Multiply）和加法（Add），对于每一次计算操作都需要在内存中读取两个数据，$x[i]$ 和 $y[i]$，最后执行一个线性操作，存储到 $y[i]$ 中，其中把加法和乘法融合在一起的操作也可以称作 FMA（Fused Multiply and Add）。

在 O(n) 的时间复杂度下，根据 n 的大小迭代计算 n 次，在 CPU 中串行地按指令顺序去执行 $AX+Y$ 程序。以 Intel Exon 8280 这款芯片为例，其内存带宽是 131 GB/s，内存的延时是 89 ns，这意味着 8280 芯片的峰值算力是在 89 ns 的时间内传输 11659 个字节（byte）数据。$AX+Y$ 将在 89 ns 的时间内传输 16 字节（C/C++中 double 数据类型所占的内存空间是 8 bytes）数据，此时内存的利用率只有 0.14%（16/11659），存储总线有 99.86% 的时间处于空闲状态。

![[Pasted image 20241011124444.png]]

不同处理器计算 $AX+Y$ 时的内存利用率，不管是 AMD Rome 7742、Intel Xeon 8280 还是英伟达 A100，对于 $AX+Y$ 这段程序的内存利用率都非常低，基本 ≤0.14%。

|                        | AMD Rome 7742 | Intel Xeon 8280 | NVIDIA A100 |
| ---- | - | --- | -- |
| Memory B/W(GB/sec)     | 204           | 131             | 1555        |
| DRAM Latency(ns)       | 122           | 89              | 404         |
| Peak bytes per latency | 24,888        | 11,659          | 628,220     |
| Memory Efficiency      | 0.064%        | 0.14%           | 0.0025%     |

由于上面的 $AX+Y$ 程序没有充分利用并发和线性度，因此通过并发进行循环展开的代码如下：

```c
void fun_axy(int n, double alpha, double *x, double *y)
{
    for (int i = 0; i < n; i += 8)
    {
        y[i + 0] = alpha * x[i + 0] + y[i + 0];
        y[i + 1] = alpha * x[i + 1] + y[i + 1];
        y[i + 2] = alpha * x[i + 2] + y[i + 2];
        y[i + 3] = alpha * x[i + 3] + y[i + 3];
        y[i + 4] = alpha * x[i + 4] + y[i + 4];
        y[i + 5] = alpha * x[i + 5] + y[i + 5];
        y[i + 6] = alpha * x[i + 6] + y[i + 6];
        y[i + 7] = alpha * x[i + 7] + y[i + 7];
    }
}
```

每次执行从 0 到 7 的数据，实现一次性迭代 8 次，每次传输 16 bytes 数据，因此同样在 Intel Exon 8280 芯片上，每 89 ns 的时间内将执行 729（11659/16）次请求，将程序这样改进就是通过并发使整个总线处于一个忙碌的状态，但是在真正的应用场景中：

- 编译器很少会对整个循环进行超过 100 次以上的展开；

- 一个线程每一次执行的指令数量是有限的，不可能执行非常多并发的数量；

- 一个线程其实很难直接去处理 700 多个计算的负荷。

由此可以看出，虽然并发的操作能够一次性执行更多的指令流水线操作，但是同样架构也会受到限制和约束。

将 $Z=AX+Y$ 通过并行进行展开，示例代码如下：

```c
void fun_axy(int n, double alpha, double *x, double *y)
{
    Parallel for (int i = 0; i < n; i++)
    {
        y[i] = alpha * x[i] + y[i];
    }
}
```

通过并行的方式进行循环展开，并行就是通过并行处理器或者多个线程去执行 $AX+Y$ 这个操作，同样使得总线处于忙碌的状态，每一次可以执行 729 个迭代。相比较并发的方式：

- 每个线程独立负责相关的运算，也就是每个线程去计算一次 $AX+Y$；

- 执行 729 次计算一共需要 729 个线程，也就是一共可以进行 729 次并行计算；

- 此时程序会受到线程数量和内存请求的约束。

### GPU 缓存机制

在 GPU 工作过程中希望尽可能的去减少内存的时延、内存的搬运、还有内存的带宽等一系列内存相关的问题，其中缓存对于内存尤为重要。英伟达 Ampere A100 内存结构中 HBM Memory 的大小是 80G，也就是 A100 的显存大小是 80G。

其中寄存器（Register）文件也可以视为缓存，寄存器靠近 SM（Streaming Multiprocessors）执行单元，从而可以快速地获取执行单元中的数据，同时也方便读取 L1 Cache 缓存中的数据。此外 L2 Cache 更靠近 HBM Memory，这样方便 GPU 把大量的数据直接搬运到 cache 中，因此为了同时实现上面两个目标，GPU 设计了多级缓存。80G 的显存是一个高带宽的内存，L2 Cache 大小为 40M，所有 SM 共享同一个 L2 Cache，L1 Cache 大小为 192kB，每个 SM 拥有自己独立的 Cache，同样每个 SM 拥有自己独立的 Register，每个寄存器大小为 256 kB，因为总共有 108 个 SM 流处理器，因此寄存器总共的大小是 27MB，L1 Cache 总共的大小是 20 MB。

![[Pasted image 20241011125544.png]]

GPU 和 CPU 内存带宽和时延进行比较，在 GPU 中如果把主内存（HBM Memory）作为内存带宽（B/W, bandwidth）的基本单位，L2 缓存的带宽是主内存的 3 倍，L1 缓存的带宽是主存的 13 倍。在真正计算的时候，希望缓存的数据能够尽快的去用完，然后读取下一批数据，此时就会遇到时延（Lentency）的问题。如果将 L1 缓存的延迟作为基本单位，L2 缓存的延迟是 L1 的 5 倍，HBM 的延迟将是 L1 的 15 倍，因此 GPU 需要有单独的显存。

假设使用 CPU 将 DRAM（Dynamic Random Access Memory）中的数据传入到 GPU 中进行计算，较高的时延（25 倍）会导致数据传输的速度远小于计算的速度，因此需要 GPU 有自己的高带宽内存 HBM（High Bandwidth Memory），GPU 和 CPU 之间的通信和数据传输主要通过 PCIe 来进行。

![[Pasted image 20241011125732.png]]



| 存储类型                               | 结构                                  | 工作原理                                                                                      | 性能                                                                                | 应用                                       |
| ---- | ----- | -------- | ------ | ---- |
| DRAM（Dynamic Random Access Memory）| 一种基本的内存技术，通常以单层平面的方式组织，存储芯片分布在一个平面上 | 当读取数据时，电荷被传递到输出线路，然后被刷新。当写入数据时，电荷被存储在电容中。由于电容会逐渐失去电荷，因此需要周期性刷新来保持数据                       | 具有较高的密度和相对较低的成本，但带宽和延迟相对较高                                                        | 常用于个人电脑、笔记本电脑和普通服务器等一般计算设备中              |
| GDDR（Graphics Double Data Rate）   | 专门为图形处理器设计的内存技术，具有较高的带宽和性能          | 在数据传输速度和带宽方面优于传统的 DRAM，适用于图形渲染和视频处理等需要大量数据传输的应用                                           | GDDR 与标准 DDR SDRAM 类似，但在设计上进行了优化以提供更高的数据传输速度。它采用双倍数据速率传输，即在每个时钟周期传输两次数据，提高了数据传输效率 | 主要用于高性能图形处理器（GPU）和游戏主机等需要高带宽内存的设备中       |
| HBM（High Bandwidth Memory）        | 使用堆叠设计，将多个 DRAM 存储芯片堆叠在一起，形成三维结构    | 堆叠设计允许更短的数据传输路径和更高的带宽，同时减少了功耗和延迟。每个存储芯片通过硅间连接（Through Silicon Via，TSV）与其他存储芯片通信，实现高效的数据传输 | 具有非常高的带宽和较低的延迟，适用于高性能计算和 AI 等需要大量数据传输的领域                                          | 主要用于高端图形处理器（GPU）、高性能计算系统和服务器等需要高带宽内存的设备中 |

不同存储和传输的带宽和计算强度进行比较，假设 HBM 计算强度为 100，L2 缓存的计算强度只为 39，意味着每个数据只需要执行 39 个操作，L1 的缓存更少，计算强度只需要 8 个操作，这个时候对于硬件来说非常容易实现。这就是为什么 L1 缓存、L2 缓存和寄存器对 GPU 来说如此重要。可以把数据放在 L1 缓存里面然后对数据进行 8 个操作，使得计算达到饱和的状态，使 GPU 里面 SM 的算力利用率更高。但是 PCIe 的带宽很低，整体的时延很高，这将导致整体的算力强度很高，算力利用率很低。

| DataLocation | Bandwidth(GB/sec) | ComputeIntensity | Latency(ns) | Threads Required |
| --- | -- | - | -- | - |
| L1 Cache     | 19,400            | 8                | 27          | 32,738           |
| L2 Cache     | 4,000             | 39               | 150         | 37,500           |
| HBM          | 1,555             | 100              | 404         | 39,264           |
| NVLink       | 300               | 520              | 700         | 13,125           |
| PCIe         | 25                | 6240             | 1470        | 2297             |

在带宽增加的同时线程的数量或者线程的请求数也需要相对应的增加，这个时候才能够处理并行的操作，每个线程执行一个对应的数据才能够把算力利用率提升上去，只有线程数足够多才能够让整个系统的内存处于忙碌的状态，让计算也处于忙碌的状态，因此看到 GPU 里面的线程数非常多。

### GPU 线程原理

GPU 整体架构和单个 SM（Streaming Multiprocessor）的架构，SM 可以看作是一个基本的运算单元，GPU 在一个时钟周期内可以执行多个 Warp，在一个 SM 里面有 64 个 Warp，其中每四个 Warp 可以单独进行并发的执行，GPU 的设计者主要是增加线程和增加 Warp 来解决或者掩盖延迟的问题，而不是去减少延迟的时间。

![[Pasted image 20241011125943.png]]

为了有更多的线程处理计算任务，GPU SMs 线程会选择超配，每个 SM 一共有 2048 个线程，整个 A100 有 20 多万个线程可以提供给程序，在实际场景中程序用不完所有线程，因此有一些线程处于计算的过程中，有一些线程负责搬运数据，还有一些线程在同步地等待下一次被计算。很多时候会看到 GPU 的算力利用率并不是非常的高，但是完全不觉得它慢是因为线程是超配的，远远超出大部分应用程序的使用范围，线程可以在不同的 Warp 上面进行调度。

|                 | Pre SM | A100    |
| --- | --- | - |
| Total Threads   | 2048   | 221,184 |
| Total Warps     | 64     | 6,912   |
| Active Warps    | 4      | 432     |
| Waiting Warps   | 60     | 6,480   |
| Active Threads  | 128    | 13,824  |
| Waiting Threads | 1,920  | 207,360 |

## 小结与思考

- GPU 与 CPU 架构差异：GPU 拥有大量并行处理单元，专为吞吐量优化，适合执行大规模并行计算任务，而 CPU 设计有少量强大计算单元，优化以降低延迟，适合处理顺序执行任务。

- GPU 并行计算原理：GPU 通过超配线程和 Warp（一组同时执行相同指令的线程）来掩盖内存延迟，利用其高吞吐量架构来提升性能，与 CPU 的低延迟设计形成对比。

- GPU 缓存机制与线程原理：GPU 采用多级缓存结构来降低对主内存的依赖，并通过大量线程超配来提高算力利用率，其中 L1 Cache、L2 Cache 和寄存器的设计有助于提升数据传输和计算效率。

----

# 为什么 GPU 适用于 AI

本节内容主要探究 GPU AI 编程的本质，首先回顾卷积计算是如何实现的，然后探究 GPU 的线程分级，分析 AI 的计算模式和线程之间的关系，最后讨论矩阵乘计算如何使用 GPU 编程去提升算力利用率或者提升算法利用率。

## Conv 卷积计算

卷积运算是深度学习中常用的操作之一，用于处理图像、音频等数据。简而言之，卷积运算是将一个函数与另一个函数经过翻转和平移后的结果进行积分。在深度学习中，卷积运算可以用来提取输入数据中的特征。

具体而言，对于输入数据 $X$ 和卷积核 $K$，卷积运算可以通过以下公式表示：

$$Y[i,j] = \sum_{m}\sum_{n} X[i+m, j+n] \cdot K[m,n]$$

其中，$Y$ 是卷积后的输出数据，$X$ 是输入数据，$K$ 是卷积核，$i$ 和 $j$ 是输出数据的索引，$m$ 和 $n$ 是卷积核的索引。

## 高性能卷积计算：img2col 原理
img2col 是一种实现卷积操作的加速计算策略。它能将卷积操作转化为 GEMM，从而最大化地缩短卷积计算的时间。

GEMM 是通用矩阵乘 (General Matrix Multiply) 的英文缩写，其实就是一般意义上的矩阵乘法，数学表达就是 C = A x B。根据上下文语境，GEMM 有时也指实现矩阵乘法的函数接口。

**卷积操作转化为GEMM好处**：
- 因为线性代数领域已经有非常成熟的计算接口（BLAS，Fortran 语言实现）来高效地实现大型的矩阵乘法，几乎可以做到极限优化。
- 将卷积过程中用到的所有特征子矩阵整合成一个大型矩阵存放在连续的内存中，虽然增加了存储成本，但是减少了内存访问的次数，从而缩短了计算时间。

img2col的原理可以用下面图来概况
![[Pasted image 20241012210408.png]]
### 1. Input Features -> Input Matrix
输入特征图一共有三个通道，我们以不同的颜色来区分
![[Pasted image 20241012210634.png]]
以蓝色的特征图为例，它是一个3 x 3的矩阵，而卷积核是一个2 x 2 的矩阵，当卷积核的滑动步长为1时，那么传统的直接卷积计算一共需要进行4次卷积操作。

现在我们把每一个特征子矩阵都排列成一个行向量，然后把这4个行向量堆叠成一个新的矩阵，就得到了Input Matrix。

### 2. Convolution Kernel -> Kernel Matrix
![[Pasted image 20241012211144.png]]
### 3. Input Matrix * Kernel Matrix = Output Matrix
在得到上述两个矩阵之后，接下来调用 GEMM 函数接口进行矩阵乘法运算即可得到输出矩阵，然后将输出矩阵通过 col2img 函数就可以得到和卷积运算一样的输出特征图。
![[Pasted image 20241012211256.png]]
## GPU 线程分级
在 AI 计算模式中，不是所有的计算都可以是线程独立的。计算中数据结构元素之间的对应关系有以下三种：

- 1）Element-wise（逐元素）：逐元素操作是指对数据结构中的每个元素独立执行操作。这意味着操作应用于输入数据结构中对应元素的每一对，以生成输出数据结构。例如，对两个向量进行逐元素相加或相乘就是将对应元素相加或相乘，得到一个新的向量。

- 2）Local（局部）：局部操作是指仅针对数据的特定子集执行的操作，而不考虑整个数据结构。这些操作通常涉及局部区域或元素的计算。例如，对图像的卷积运算中元素之间是有交互的，因为它仅影响该区域内的像素值，计算一个元素往往需要周边的元素参与配合。

- 3）All to All（全对全）：全对全操作是指数据结构中的每个元素与同一数据结构或不同数据结构中的每个其他元素进行交互的操作。这意味着所有可能的元素对之间进行信息交换，产生完全连接的通信模式，一个元素的求解得到另一个数据时数据之间的交换并不能够做到完全的线程独立。全对全操作通常用于并行计算和通信算法中，其中需要在所有处理单元之间交换数据。

![[Pasted image 20241012211434.png]]

以卷积运算为例解释在卷积运算中局部内存数据如何与线程分层分级配合工作的，以下是处理一个猫的图像数据，基本过程如下：

- 1）用网格（grid）覆盖图片，将图片切割成多个块（block）；
- 2）取出其中的一个块进行处理，同样也可以将这个块再进行分割；
- 3）块中线程（Threads）通过本地数据共享来进行计算，每个像素点都会单独分配一个线程进行计算。

![[Pasted image 20241012211521.png]]

因此可以将大的网格表示为所有需要执行的任务，小的切分网格中包含了很多相同线程数量的块，块中的线程数独立执行，可以通过本地数据共享实现同步数据交换。

![[Pasted image 20241012211614.png]]


前面的章节讲到 GPU 的并行能力是最重要的，并行是为了解决带宽的时延问题，而计算所需要的线程数量是由计算复杂度决定的，结合不同数据结构对数据并行的影响：

- 1）逐元素（Element-wise）每增加一个线程意味着需要对数据进行新一次加载，由于在 GPU 中线程是并行的，因此增加线程的数量并不能对实际运算的时延产生影响，数据规模在合理范围内增大并不会影响实际算法的效率。

- 2）局部（Local）数据结构在进行卷积这类运算时由于线程是分级且并行的，因此每增加一个线程对于数据的读取不会有较大影响，此时 GPU 的执行效率与 AI 的计算模式之间实现了很好的匹配，计算强度为 O(1)。

- 3）全对全（All to All）一个元素的求解得到另一个数据时数据之间的交换并不能够做到完全的线程独立，此时计算强度会随着计算规模的增加线性增加，All to All 操作通常需要进行大量的数据交换和通信。

![[Pasted image 20241012211717.png]]

## 计算强度

由于 AI 计算可以看作是矩阵相乘，两个矩阵相乘得到一个新的矩阵。假设有两个矩阵 $A$ 和 $B$，它们的维度分别为 $m \times n$ 和 $n \times p$，则它们可以相乘得到一个新的矩阵 $C$，其维度为 $m \times p$。其计算公式为：

$$c_{ij} = \sum_{k=1}^{n} a_{ik} \cdot b_{kj}$$

其中，$a_{ik}$ 是矩阵 $A$ 中第 $i$ 行第 $k$ 列的元素，$b_{kj}$ 是矩阵 $B$ 中第 $k$ 行第 $j$ 列的元素。通过对所有可能的 $k$ 值求和，可以得到 $C$ 中的每一个元素。

![[Pasted image 20241012211758.png]]

计算强度（Arithmetic Intensity）是指在执行计算任务时所需的算术运算量与数据传输量之比。它是衡量计算任务的计算密集程度的重要指标，可以帮助评估算法在不同硬件上的性能表现。通过计算强度，可以更好地理解计算任务的特性，有助于选择合适的优化策略和硬件配置，以提高计算任务的性能表现。计算强度的公式如下：

$$\text{计算强度} = \frac{\text{算术运算量}}{\text{数据传输量}}$$

其中，算术运算量是指执行计算任务所需的浮点运算次数，数据传输量是指从内存读取数据或将数据写入内存的数据传输量。计算强度的值可以用来描述计算任务对计算和数据传输之间的依赖关系：

-  高计算强度：当计算强度较高时，意味着算术运算量较大，计算操作占据主导地位，相对较少的时间用于数据传输。在这种情况下，性能优化的重点通常是提高计算效率，如优化算法、并行计算等。

-  低计算强度：当计算强度较低时，意味着数据传输量较大，数据传输成为性能瓶颈。在这种情况下，性能优化的关键是减少数据传输、优化数据访问模式等。

对于一个 $N \times N$ 矩阵乘法操作，可以计算其计算强度（Arithmetic Intensity）。

1.  **算术运算量**：对于两个 $N \times N$ 的矩阵相乘，总共需要进行 $N^3$ 次乘法和 $N^2(N-1)$ 次加法运算。因此，总的算术运算量为 $2N^3 - N^2$。

2.  **数据传输量**：在矩阵乘法中，需要从内存中读取两个输入矩阵和将结果矩阵写回内存。假设每个矩阵元素占据一个单位大小的内存空间，则数据传输量可以估计为 $3N^2$，包括读取两个输入矩阵和写入结果矩阵。

因此，矩阵乘法的计算强度可以计算为：

$$\text{Arithmetic Intensity} = \frac{2N^3 - N^2}{3N^2}≈O(N)$$

因此矩阵乘法的计算强度用时间复杂度表示为 $O(N)$，随着相乘的两个矩阵的维度增大，算力的需求将不断提高，需要搬运的数据量也将越大，算术强度也随之增大。

计算强度和矩阵维度的大小密切相关，图中蓝线表示矩阵乘法的算术强度随着矩阵的大小增大线性增加，橙色的线表示 GPU FP32 浮点运算的计算强度，橙色线与蓝色线的交点表示当计算单元充分发挥计算能力时矩阵的大小约为 50，此时满足整个 GPU FP32 的计算强度，实现理想情况下计算和搬运数据之间的平衡。

当矩阵大小不断增加时，GPU 中的内存会空闲下来（内存搬运越来越慢导致内存刷新变慢），GPU 需要花费更多的时间执行矩阵计算，因此 AI 计算需要找到一个更好的平衡点去匹配更大的矩阵计算和计算强度。

图中红色的线是英伟达 GPU 采用 Tensor Core 专门对矩阵进行计算，很大程度上提高了计算强度，使得内存的搬运能够跟得上数据运算的速度，更好地平衡了矩阵维度和计算强度之间的关系。

![[Pasted image 20241012212001.png]]

> FP32 和 FP64
> 
> GPU 计算中的 FP32 和 FP64 分别代表单精度浮点运算和双精度浮点运算，主要区别在于精度和计算速度。FP32 使用 32 位存储单精度浮点数，提供较高的计算速度，但在处理非常大或非常小的数字时可能存在精度损失。相比之下，FP64 使用 64 位存储双精度浮点数，提供更高的精度，但计算速度通常较慢。
> 
> 在实际应用中，选择 FP32 还是 FP64 取决于任务的需求。如果任务对精度要求不高并且需要较高的计算速度，则可以选择 FP32。但如果任务对精度要求非常高，则需要选择 FP64，尽管计算速度可能会受到影响。

在新增了 Tensor Core 之后对不同存储和传输的带宽和计算强度进行比较，采用 L1 缓存的计算强度为 32，采用 L2 缓存的计算强度是 156，因此需要考虑如何搭配多级缓存和 Tensor Core(张量核心)，使得张量核心在小矩阵或大矩阵计算中都能够更高效地执行运算。

| Data Location | Bandwidth(GB/sec) | Compute Intensity | Tensor Core |
| --- | --- | --- | --- |
| L1 Cache | 19,400 | 8 | 32 |
| L2 Cache | 4,000 | 39 | 156 |
| HBM | 1,555 | 100 | 401 |
| NVLink | 300 | 520 | 2080 |
| PCIe | 25 | 6240 | 24960 |

> 张量核心（Tensor core）
> 
> Tensor Core 是英伟达推出专门用于深度学习和 AI 计算的硬件单元。Tensor Core 的设计旨在加速矩阵乘法运算，这在深度学习中是非常常见的计算操作。Tensor Core 主要有以下特点和优势：
> 
> 1. 并行计算能力：Tensor Core 能够同时处理多个矩阵乘法运算，从而大幅提高计算效率。
> 
> 2. 混合精度计算：Tensor Core 支持混合精度计算，即同时使用浮点 16 位（half-precision）和浮点 32 位（single-precision）数据类型进行计算，以在保证计算精度的同时提高计算速度。
> 
> 3. 高性能计算：Tensor Core 具有非常高的计算性能，能够快速处理大规模的神经网络模型和数据集。
> 
> 4. 节能优势：由于其高效的并行计算和混合精度计算能力，Tensor Core 在相同计算任务下通常能够比传统的计算单元更节能。
> 

当 Tensor Core 在 L1 缓存、L2 缓存和 HBM 存储位置的不同将影响理想计算强度下矩阵的维度大小，每种存储和矩阵的计算强度分别对应一个交叉点，由此可以看出数据在什么类型的存储中尤为重要，相比较 FP32 和 FP64 对计算强度的影响更为重要。当数据搬运到 L1 缓存中时可以进行一些更小规模的矩阵运算，比如卷积运算，对于 NLP（Natural Language Processing）中使用的 transformer 结构，可以将数据搬运到 L2 缓存进行计算。因为数据运算和读取存在比例关系，如果数据都在搬运此时计算只能等待，导致二者不平衡，因此找到计算强度和矩阵大小的平衡点对于 AI 计算系统的优化尤为重要。

![[Pasted image 20241012212430.png]]

## 小结与思考

- GPU 的并行处理能力强，适合执行深度学习中的卷积运算，这些运算本质上是大规模的矩阵乘法，而 GPU 专为这类任务优化。

- GPU 的线程分级结构能够与 AI 计算中的不同数据结构对应关系相匹配，通过线程超配和 Warp 执行单元来提高计算效率并掩盖内存延迟。

- 计算强度是衡量 AI 计算性能的关键指标，GPU 通过优化计算强度，如引入 Tensor Core，来提升 AI 计算中矩阵乘法的性能，实现计算与数据传输之间的有效平衡。

