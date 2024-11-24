# 摘要
本文回顾了类脑计算三大类基础软件的现状，即神经形态芯片的工具链、软件模拟框架以及集成脉冲神经网络（SNN）和深度神经网络（DNN）的框架。随后，我们指出，“通用”的分层和软硬件分离的基础软件框架将有利于（计算）神经科学和脑启发智能领域。“通用”的概念是指软件和硬件的分离，并支持计算机科学和神经科学相关研究的整合。

# 1. 介绍
对于神经科学和计算神经科学研究人员来说，他们关注的类脑计算基础软件的特点包括高效、合理地分析和解释实验数据以建立计算神经模型（从单个神经元到大规模神经回路），并利用生物可行的可塑性规则对其进行优化（类脑计算中的主要计算范式是脉冲神经网络，简称SNN），以增进对生物智能基础设施和底层机制的理解和认识。

# 2. Three types of the basic software for brain-inspired computing
类脑计算中应用最广泛的计算模型是脉冲神经网络，其主要计算过程如下：
- 首先，需要求解膜电位的动力学方程（通常为微分方程），得到膜电位的当前值；
- 其次，如果膜电位达到阈值电压，神经元就会发出脉冲，必要时电压会在复位电压上停留一段时间；
- 第三，需要根据联结原理将发出的脉冲传递给目标突触；
- 最后，接收到脉冲的突触根据突触模型更新目标神经元的内部状态。
神经元模型定义了膜电位的动态以及其他内部状态。神经元模型有很多，例如 leakyintegrate-and-fre (LIF) (Burkitt 2006)、Izhikevich (2003)、Hodgkin 和 Huxley (1990) 等等 (Brette et al. 2007)。不同的模型提供不同水平的生物现实和计算效率。


![[Pasted image 20241112192050.png]]


The first is to develop neuromorphic chips and provide their own software toolchains, including IBM's **TrueNorth** chip (Akopyan et al. 2015), Intel's **Loihi** chip (Davies et al. 2018), **SpiNNaker** (Brown et al. 2018), and **BrainScaleS** system (Schemmel et al. 2012) (both are supported by the Human Brain Project ), and Stanford University’s **NeuroGrid** (Benjamin et al. 2014), etc.

The second is SNN simulation frameworks derived from the field of computational neuroscience, with the goal of understanding biological systems, including **NEST** (Gewaltig and Diesmann 2007), **NEURON** (Carnevale and Hines 2006), **Brain2** (Stimberg et al. 2014), **GeNN** (Yavuz et al. 2016), etc.

该类工作的特点是支持更为细致的神经元活动动态过程的描述，支持多种类型的突触模型和多种类型的突触可塑性，从而能够细致地模拟生物神经网络，但要求使用者具备一定的计算神经科学基础。

第三类是近年来兴起的将SNN的表示/计算特性融入到广泛使用的深度学习开发框架。

# 3. Neuromorphic chips and the toolchains of them
**TrueNorth** (Akopyan et al. 2015) 是 IBM 推出的一款同步异步混合全数字神经形态芯片，由 5.4 亿个晶体管组成，提供 4096 个神经突触核，每个核包含 256 个神经元和 64000 个突触，总共可支持约 100 万个神经元和 2.56 亿个突触。该芯片峰值计算性能为 58 GSOPS (每秒千兆突触操作)，峰值计算功耗为 400 (GSOPS/W)。TrueNorth 使用的神经元模型是简化的 LIF 模型 (Cassidy et al. 2013)。为了充分挖掘 TrueNorth 的潜力，IBM 开发了一系列专用工具。最重要的工具是 **Compass**（Preissl 等人，2012 年），它是 TrueNorth 架构的模拟器，以及 **Corelet**（Amir 等人，2013 年），这是一种新的编程范式，由程序抽象、面向对象语言、库和端到端编程环境组成。这些工具可以直接使用 TrueNorth 使用的模型和连接模式进行编程，然后可以编译程序并将其映射到芯片上。

**Loihi** (Davies et al. 2018) 是英特尔推出的支持突触可塑性策略的异步神经形态芯片，提供对分层连接、树突区室、突触延迟、可编程突触学习规则等多项重要特性的支持。Loihi 是一款网格状多核脉冲神经网络处理器，每颗芯片包含 128 个神经形态核心、3 个 X86 处理器核心、一个用于扩展的片外接口和一个用于消息传输的异步片上网络。它总共有 20.7 亿个晶体管和 33 MB SRAM。每个神经形态核心可以通过时分复用模拟 1024 个神经部分（树突或胞体）。英特尔实验室还提供了 Loihi 工具链 (**Lin** et al. 2018)，它由三部分组成：用于指定 SNN 基于 Python 的 API、编译器和运行时，用于在 Loihi 和多个目标平台（Loihi 硅片、FPGA 和功能模拟器）上构建和执行 SNN。

**SpiNNaker**（脉冲神经网络架构）（Brown 等，2018）是一个大规模并行计算平台，旨在实时模拟大规模脉冲神经网络。它计划使用 65,536 个 MPSoC（Multi-Processor System-on-Chip）芯片（共计 1,179,648 个处理器）模拟 10 亿个神经元和 1 万亿个突触。MPSoC 芯片采用 GALS（全局异步局部同步）设计方案，每个芯片集成 18 个同构的 ARM968 整数处理器核、路由器、DMA 控制器和其他外围设备。由于其计算由 ARM 内核完成，因此提出了几个有助于模拟其他软件中描述的模型的软件包，例如用于 **PyNN**（Davison 等人，2009 年）的 **SPyNNaker**（Rhodes 等人，2018 年）和用于 **Nengo**（Bekolay 等人，2014 年）的 **nengo_spinnaker**。