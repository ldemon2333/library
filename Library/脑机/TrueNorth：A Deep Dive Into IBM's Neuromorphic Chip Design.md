https://open-neuromorphic.org/blog/truenorth-deep-dive-ibm-neuromorphic-chip-design/

# Why do we want to emulate the brain?
 “The brain is much powerful than any AI machine when it comes to cognitive tasks but it runs on a **10W** power budget!”. This is absolutely true: neurons in the brain communicate among each other by means of **spikes**, which are short voltage pulses that propagate from one neuron to the other. The average spiking activity is estimated to be around **10Hz** (i.e. a spike every 100ms). This yields **very low processing power consumption**, since the activity in the brain results to be **really sparse** (at least, this is the hypothesis).

How can the brain do all this? There are several reasons (or hypotheses, I should say):

- the 3D connectivity
- extremely lower power operation

Hence, IBM decided to try to emulate the brain with **TrueNorth**, a 4096 cores chpi packing 1 million neurons and 256 million synapases.

# Introduction
The TrueNorth design has been driven by seven principles.
## Purely event-driven architecture
该体系结构是一个纯事件驱动的体系结构，being Globally Asynchronous Locally Synchronous (GALS) ，在同步核之间有一个完全异步的互连结构。

![[Pasted image 20241112195030.png]]

一般来说，在 GALS 架构中，有一组处理元件(PE)通过全局时钟同步。PE 中的本地时钟对于每个 PE 可能是不同的，因为每个 PE 可能以不同的速度运行。当两个不同的时钟域需要接口时，它们之间的通信是有效的异步通信: 为了保证正确的全局操作，必须在它们之间实现握手协议。

TrueNorth 和 SpiNNaker 一样，没有全局时钟: PE 是神经突触核心，通过一个完全异步的网络相互连接。通过这种方式，芯片操作是事件驱动的，因为只有当有峰值(和其他类型的事件)要传输时，网络才会被激活。

## Low power operation
所采用的 CMOS 工艺是一种低功耗的工艺，其目标是最大限度地减少静态功耗。技术节点为28nm CMOS。

## Massive parallelism
由于大脑是一个大规模并行处理机结构，使用了1000亿个神经元，每个神经元有大约10000个突触，并行性是 TrueNorth 的一个关键特征: 芯片使用了100万个神经元和2.56亿个突触，通过互连4096个核心，每个核心模拟256个神经元和64000个突触。

## Real time operation
作者声称实时操作，这意味着1毫秒的全局时间同步，也就是说，神经元每毫秒更新和尖峰。

## Scalable design
该体系结构是可扩展的: 多个核可以放在一起，由于时钟信号只分布在本地，在核心结构中，现代 VLSI 数字电路的全局时钟信号倾斜问题不会影响 TrueNorth。

## Error tolerance
设计中采用了冗余技术，特别是在存储电路中，使芯片具有容错性。


# One-to-one correspondence between software and hardware
当使用 IBMTrueNorth 应用程序设计软件时，芯片操作与软件操作完全对应。

设计一个异步电路是一个非常困难的任务，因为没有超大规模集成电子设备可用于这种设计，因此，TrueNorth 的设计者决定使用传统的电子设备进行同步核心设计和定制设计工具以及异步互连结构的流程。

# Architecture
## Who's Von Neumann?
The TrueNorth chip is not a Von Neumann machine!
![[Pasted image 20241112201957.png]]

在冯诺依曼机器中，就像上面描述的处理单元与存储单元分离，存储单元同时存储数据和指令。处理器从内存中读取指令，对其进行解码，从相同的内存中检索需要操作的数据，然后执行指令。

原则上，神经形态学芯片是一个 in-memory computing 架构: 在这里，没有中央存储器和中央处理单元，但是存储和计算电路是分布式的，也就是说，我们有许多小的存储器和小的计算单元，如下图所示。
![[Pasted image 20241112202129.png]]

有两个优点：
- **lower energy consumption** associated to memory accesses. The main power consumption involved in a memory access is the one corresponding to the **bus data movement**. A data bus, simplifying, is a big RC circuit, 每次我们让它上面的信号发生变化，我们就会消耗大量的能量来驱动这个等效电路。我们可以很容易地推断出电阻和电容的值与总线长度成正比！因此，通过将处理单元(PE)和内存放在一起，可以减少数据移动功耗。
- **lower latency** associated to memory accesses. A big RC circuit (i.e. a long bus) is also slower than a short one (i.e. the time constant associated to the equivalent circuit is larger); 因此，通过缩短其长度，我们也减少了读取或写入数据到内存所需的时间。
- **high parallelism**。PE 可以并行工作，因为每个 PE 都可以独立于其他 PE 访问自己的数据。

然而，天下没有免费的午餐: 这种架构有一个很大的缺点，**area occupation** of the circuit. In a Von Neumann architecture, the **memory density** is **higher**: in VLSI circuits, the larger the memory, the higher the number of bits you can memorise per squared micrometer; hence, the total area occupied by the memories in an in-memory computing architecture is larger than the one corresponding to a Von-Neumann circuit. Moreover, you have multiple PEs performing the same operation on different data (this kind of architecture is also called Single Instruction Multiple Data (SIMD)); with a central memory, you can use a **single PE** to perform the same operations, saving lots of chip area at the expense of performance, since you cannot perform operations in parallel.

---
这段话讨论了 in-memory computing architecture 相对于传统的**冯·诺依曼架构（Von Neumann architecture）的一个主要劣势：**电路的面积占用问题**。

1. **存储密度的比较**：在冯·诺依曼架构中，存储密度较高。通常在超大规模集成电路（VLSI）中，存储器的密度越高，每平方微米可以存储的比特数量越多。相比之下，存内计算架构中的存储面积较大，即为相同容量的存储器，需要占用更多的电路面积。

2. **计算单元和存储的分布**：存内计算架构通常会有多个**处理单元（Processing Elements, PEs）**同时执行相同的操作，但每个单元处理不同的数据，这种方式称为**单指令多数据（SIMD）**。与之相对，冯·诺依曼架构使用中央存储器，并且可以用一个PE完成所有操作。这种架构节省了大量芯片面积，但代价是**牺牲性能**，因为它无法实现并行操作。

总结起来，这种架构的优势是并行处理能力，但劣势是增加了芯片面积。

---
## Memory and computation co-location: emulating the brain

![[Pasted image 20241112203603.png]]

一个神经元由不同的部分组成，如上图所示。树突从细胞体分支出来，也称为 soma ，细胞核就位于这里。然后，有一个长的沟通通道称为轴突，which ends in the pre-synaptic terminal，可以有多个分支。

![[Pasted image 20241112203722.png]]

Dendrites branch out from the soma. Their function is to **receive information** from other neurons. Some dendrites have small protrusions called **spines** that are important for communicating with other neurons.

![[Pasted image 20241112203808.png]]

The soma is where the **computation** happens. This is where the membrane potential is built up, by ions exchange with the environment and other neurons.

![[Pasted image 20241112203849.png]]

The axon is the communication channel of the neuron. It is attached to the neuron through the **axon hillock**; at the end of the axon, we find the pre-synaptic terminals, which are the “pins” used to connect to the **post-synaptic** terminal of other neurons. These connections are called **synapses**.

![[Pasted image 20241112203956.png]]

轴突 terminates at the pre-synaptic terminal or terminal **bouton**. The terminal of the pre-synaptic cell forms a synapse with another neuron or cell, known as the post-synaptic cell. When 动作电位 reaches the pre-synaptic terminal, the neuron releases 神经递质 into the synapse. The neurotransmitters act on the post-synaptic cell. Therefore, neuronal communication requires both an electrical signal (the action potential) and a chemical signal (the neurotransmitter). Most commonly, pre-synaptic terminals contact dendrites, but terminals can also communicate with cell bodies or even axons. Neurons can also synapse on non-neuronal cells such as muscle cells or glands.

The terms pre-synaptic and post-synaptic are in reference to which neuron is releasing neurotransmitters and which is receiving them. Pre-synaptic cells release neurotransmitters into the synapse and those neurotransmitters act on the post-synaptic cell.

轴突 transmit an **action potential**, which is the famous spike! This results in the release of chemical neurotransmitters to communicate with other cells. 

轴突产生动作电位，是一个spike，传递到自己的 pre-synaptic terminal，然后会导致pre-synaptic terminal 释放神经递质给其他神经元。

## From biology to silicon
In a neuromorphic chip, hence, memory and computational units are **co-located**. The neuron constitutes the computational unit, **while the synapses weights and the membrane potential are the data on which the neuron operates.** The chip is programmed by deciding **which neurons are connected to which**; hence, we do not write instructions to be executed to a memory, but we program the neurons interconnections and parameters!

内存和计算单元是同地协作的，神经元构成计算单元。

![[Pasted image 20241112205204.png]]

In the figure above, the logical representation of a TrueNorth core is reported. Consider the sub-figure on the left: on the right, the post-synaptic neurons are represented with a triangular shape, and these are connected to some neurons on the left, which outputs are represented by those AND-gate-shaped objects. It is an example of **fully-connected** layer in artificial neural networks.

In the sub-figure on the right, the logic implementation of this layer is depicted. Input spikes are collected in **buffers**: since in the chip the spikes are evaluated periodically (there is a **clock tick** distributed every 1ms), we need to store these until we can evaluate them; for these reason, we need local storage. Which spike is delivered to which neuron is determined by the connectivity, here illustrated through a **crossbar**: a dot on a wire represents a connection between the corresponding post-synaptic neuron dendrite (vertical wires) and the pre-synaptic neuron axon terminals (horizontal wires); this connection is the **synapse**, and its “strength” is the synapse **weight**.

When the clock tick arrives, the neurons process the incoming spikes and, if they have to, they spike and send these to the network of neurons. We can have local connections (i.e. the spikes are redistributed in the chip) or global connections (the spikes are delivered outside the chip through the Network-on-Chip (NoC).

There are some additional blocks, such as the Pseudo Random Number Generator (PRNG), that are used for more complex features, such as **stochastic spike integration**, **stochastic leakage**, **stochastic thresholds**, and so on.

## Neuron model

