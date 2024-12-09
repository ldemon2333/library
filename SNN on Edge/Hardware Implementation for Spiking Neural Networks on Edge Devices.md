https://link.springer.com/chapter/10.1007/978-3-031-18034-7_13#Abs1

# 3. Hardware Accelerators for SNNs on the Edge
SNN hardware implementations on the edge devices 的 three metrics:
- energy consumption
- hardware resources 
- response time delay

讨论 2 点：
- Discuss the approaches to reduce these costs in the existing hardware acceletors for SNN.
- Furthermore, we will present a survey of the existing low-power, flexible SNN processors in the edge-IoT domain, which have been integrated with high-level APIs to enable fast prototyping of the SNN architectures.

## 3.1 Optimizations that Exploit the Temporal Sparsity of SNN
Therefore, many existing neuromorphic hardware also exploit the temporal sparsity of the spikes in SNNs to performe the computation in an event-driven manner. ==许多现有的神经形态硬件 [3, 23, 32–37] 也利用 SNN 中脉冲的时间稀疏性以事件驱动的方式执行计算。==在这种设计中，突触后神经元的膜电位仅在突触前神经元的脉冲到达时更新。在空闲时间内，硬件实现只需要静态功率来保留存储单元中的数据 [7]。突触权重更新也可以在模拟在线学习的芯片上以事件驱动的方式执行 [23, 37, 38]。因此，模拟 SNN 的硬件实现的能耗可以低于模拟 ANN 的硬件实现的能耗 [1]。

利用 SNN 中脉冲的时间稀疏性以事件驱动的方式进行计算 -> 三点：
- the membrane potentials of the postsynaptic neurons are only updated upon the arrival of a spike from the presynaptic neurons.
- During the idle time ,the hardware implementation only requires the static power to retain the data in the memory cells.
- The synaptic weight update may also be performed in an event-driven manner on the chips that emulate the online learning.
结果是：模拟 SNN 的硬件实现的能耗可以低于模拟 ANN 的硬件实现的能耗 

==通过消除耗电的全局时钟信号 [7, 31]，还可以优化边缘设备上 SNN 硬件实现的能耗。==针对边缘物联网领域的应用，提出了一种小型、低功耗神经形态处理器，称为 ==μBrain [7]==，如表 1 所示。μBrain 的硬件实现利用多相振荡器中的延迟单元执行异步脉冲通信和事件驱动的神经元计算。当脉冲到达神经元核心时，设置多相振荡器的振荡周期。神经元核心的脉冲接收单元接收信号并触发事件驱动的膜电位积累。神经元核心的脉冲传输遵循 AER 协议 [39]，该协议是在早期的工作 [10, 23, 31] 中提出的。在 AER 通信方案中（参见图 3），脉冲事件被编码到包含神经元地址的轻量级数据包中，并通过异步总线传输到目标神经元核心。由于脉冲传输是异步、轻量级和事件驱动的，因此它在 SNN 的多核硬件架构上会产生低功耗和通信延迟。==因此，AER 协议已在大型神经形态芯片 [31] 以及边缘 SNN 应用的低功耗、小占用空间的硬件实现中得到采用 [7、10、23]。（边缘SNN的通信协议）==

![[Pasted image 20241206115207.png]]
![[Pasted image 20241206120929.png]]

## 3.2 Data and Memory-Centric Architectures
==in-memory 和 near-memory 架构的两种优化方式。

传统的计算系统是基于冯·诺依曼架构开发的，该架构将内存存储元件和计算单元分开。然而，数据移动的成本（延迟和能量）已成为使用这种架构的以数据为中心的应用程序的重大瓶颈。这促使神经形态硬件向非冯·诺依曼架构（即 near-memory 计算和 in-memory 计算）转变[41]。在 near-memory 计算范式中，内存单元分布在数据被使用的处理元件附近[3,33,38]，如图4所示。==然而，in-memory 计算架构利用内存设备的物理特性在内存中执行神经元计算。==在本小节中，我们将讨论减少 in-memory 计算架构中数据移动的技术，然后讨论可用于能源受限的边缘计算应用的内存计算设计。

![[Pasted image 20241206121503.png]]

==在 near-memory 计算架构中，the data movement on the neuromorphic hardware can be optimized by spike bundling or batch processing the membrane potential accumulation and synaptic weight update.可以通过脉冲捆绑或批处理膜电位积累和突触权重更新来优化神经形态硬件上的数据移动 [3, 32–35]。== ==By delaying and dynamically grouping the spike events to be transmitted from the presynaptic to the postsynaptic neurons over multiple time steps, the communication time, number of operations, and memory access on the hardware implementation may be reduced. 通过延迟和动态分组多个时间步骤内从突触前神经元传输到突触后神经元的脉冲事件 [32, 35]，可以减少硬件实现上的通信时间、操作次数和内存访问。但是，需要根据网络的脉冲活动确定脉冲传输的批处理大小和最大时间延迟，以实现合理的能量-精度权衡 [35]。LIF 神经元中的漏电流的计算也可以推迟到下一个膜电位积累，而不是在每个时间步骤中执行 [3]。这减少了操作数量，还节省了访问潜在内存的时间和能量，尤其是当需要从片外存储加载时。此外，通过减少片上学习期间突触权重更新的次数，可以节省计算和内存访问。突触权重修改可以每隔几个时间步骤 [33, 42] 计算一次，或者当有新的传入尖峰时 [34]。还可以考虑使用近似值来减少移动设备上 DNN 的计算繁重操作（例如卷积和乘法）的数量和通信成本 [43–45]。==

==in-memory 主要利用非易失性存储器设备进行内存中的计算和数据的存放==
==in-memory 计算方法利用内存设备的物理特性和组织来执行数据存储中的膜电位积累和突触权重更新。==随着内存和计算单元之间的数据移动最小化，用于数据传输的能量和硬件也减少了，尽管代价可能是内存设计更复杂。==忆阻设备==以交叉阵列架构组织，如图 5 所示，其电导率根据突触权重值设置。输入数据表示为行上的电压，而输出数据则根据交叉阵列列上产生的电流进行测量。ANN 和 SNN 的输入和突触权重的乘法是基于欧姆定律进行的，该定律规定了电压、电流和电导之间的关系 [41]。==新兴的非易失性存储器设备，例如：相变存储器 (PCM) [46]、电阻式随机存取存储器 (RRAM) [47]、自旋转移力矩磁性 RAM (STT-RAM) [48] 和磁性 skyrmions [49] 被用于执行 SNN 的 in-memory 计算，从而实现低能耗、快速响应时间和小占用空间。这些 in-memory 计算设计可以用作 SNN 开发栈的后端，其中包括从高级 API 到设备级实现的组件，我们将在下文中讨论。

![[Pasted image 20241206124123.png]]

## 3.3 Flexible Hardware Architectures for SNN on the Edge
尽管与 HPC 集群上的 SNN 实现相比，针对边缘物联网领域的神经形态硬件消耗的功率和面积较少，但对于不熟悉硬件开发过程的神经科学家来说，它们中的大多数都缺乏灵活性和易用性。与能够模拟各种 SNN 架构和神经元模型的 GPU 上的 SNN 模拟器 [29, 30] 相比，大多数神经形态硬件模拟具有固定配置的 SNN [51]。因此，人们一直在努力弥合硬件开发和高级原型设计之间的差距，以便在边缘设备上快速开发和评估 SNN 模型 [8, 51, 55–61]。

==Caspian 是一个神经形态开发平台，由一个低功耗神经形态处理器和高级 API 组成，可用于为边缘物联网应用模拟小规模 SNN [51]。==使用 Python API 在软件中定义网络大小、连接性和超参数（例如触发阈值、泄漏时间常数、突触延迟和轴突延迟）。==此外，神经形态计算框架 TeNNLab [55] 与 Caspian 集成，以弥合软件应用程序与边缘设备上的底层硬件实现之间的差距。==

与 TeNNLab 类似，有几种神经计算框架提供高级接口来配置 SNN 架构和超参数，以便在边缘的各种神经形态硬件上进行模拟 [57–61]。==还提出了节能硬件实现，作为边缘设备上几个高级接口的后端 [8, 50, 62]。==开发了一种 SNN 协处理器，它执行从板上主处理器发出的一组定制命令，以便能够对具有任意数量神经元、突触和层的 SNN 架构进行编程。还可以使用定制命令配置 SNN 的超参数，而无需重新编程硬件实现 [38]。表 2 列出了针对边缘物联网应用的选定灵活神经形态硬件。这些硬件实现能够使用各种学习算法执行片上学习，并消耗少量片上内存。
![[Pasted image 20241206125423.png]]

[3] Roy A, Venkataramani S, Gala N, Sen S, Veezhinathan K, Raghunathan A (2017) [[A programmable event-driven architecture for evaluating spiking neural networks]]. In: ISLPED, IEEE, Piscataway, pp 1–6. https://doi.org/10.1109/ISLPED.2017.8009176  

[7] Stuijt J, Sifalakis M, Yousefzadeh A, Corradi F (2021) [[μBrain An event-driven and fully synthesizable architecture for spiking neural networks]]. Front Neurosci 15:538. https://doi.org/10.3389/fnins.2021.664208

[10] Buhler FN, Brown P, Li J, Chen T, Zhang Z, Flynn MP (2017) A 3.43 TOPS/W 48.9 pj/pixel50.1 nj/classification 512 analog neuron sparse coding neural network with on-chip learning and classification in 40 nm CMOS. In: IEEE Symp. VLSI Circuits, IEEE, pp C30–C31. https://doi.org/10.23919/VLSIC.2017.8008536

[23] Frenkel C, Lefebvre M, Legat JD, Bol D (2018) A 0.086-mm2 12.7-pJ/SOP 64k-synapse 256-neuron online-learning digital spiking neuromorphic processor in 28-nm CMOS. IEEE TransBiomed Circuits Syst 13(1):145–158. https://doi.org/10.1109/TBCAS.2018.2880425

[31] Akopyan F, Sawada J, Cassidy A, Alvarez-Icaza R, Arthur J, Merolla P, Imam N, Nakamura Y,Datta P, Nam GJ et al. (2015) TrueNorth: Design and tool flow of a 65 mW 1 million neuron programmable neurosynaptic chip. IEEE TCAD 34(10):1537–1557. https://doi.org/10.1109/TCAD.2015.2474396

[51] Mitchell JP, Schuman CD, Patton RM, Potok TE (2020) Caspian: a neuromorphic development platform. In: NICE Workshop, pp 1–6. https://doi.org/10.1145/3381755.3381764


