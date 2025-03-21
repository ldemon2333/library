# Abstract
如今，云平台一直在部署硬件加速器，如神经处理单元 (NPU)，以支持机器学习 (ML) 推理服务。为了最大限度地提高资源利用率，同时确保合理的服务质量，一种自然的方法是虚拟化 NPU，以便为多租户 ML 服务实现高效的资源共享。然而，为现代云平台虚拟化 NPU 并不容易。这不仅是因为缺乏对 NPU 硬件的系统抽象支持，还因为缺乏架构和 ISA 支持，无法为虚拟化 NPU 实现细粒度的动态运算符调度。

我们提出了 Neu10，一个整体的 NPU 虚拟化框架。我们研究了整个软件和硬件堆栈中的 NPU 虚拟化技术。Neu10 包括 (1) 一个灵活的 NPU 抽象，称为 vNPU，它支持对物理 NPU (pNPU) 中的异构计算单元进行细粒度虚拟化；(2) vNPU 资源分配器，支持按需付费计算模型和灵活的 vNPU 到 pNPU 映射，以提高资源利用率和成本效益；(3) 现代 NPU 架构的 ISA 扩展，用于促进多个 vNPU 的细粒度张量运算符调度。我们基于生产级 NPU 模拟器实现 Neu10。我们的实验表明，与最先进的 NPU 共享方法相比，Neu10 将 ML 推理服务的吞吐量提高了 1.4 倍，并将尾部延迟降低了 4.6 倍，同时平均将 NPU 利用率提高了 1.2 倍。

# Intro
To accelerate these ML services, cloud platforms have employed hardware acceletors like neural processing units (NPUs) as the mainstream compute engine.

A typical NPU device is a peripheral board with multiple NPU chips, and each chip has multiple NPU cores. Each NPU core has matrix engines (MEs) that leverage systolic arrays to perform matrix multiplications and vector engines (VEs) for generic vector operations.

A common approach to using NPUs in cloud platforms is to assign an entire NPU chip to a single ML application instance in a virtual machine (VM) or container via PCIe pass-through.

在云平台中使用NPU时，一个常见的方法是通过PCIe直通（PCIe pass-through）技术，将整个NPU芯片分配给虚拟机（VM）或容器中的单个机器学习（ML）应用实例。下面详细解释这一做法及其原因：

### 1. 什么是PCIe直通？

- **直接设备访问：**  
    PCIe直通技术允许将物理硬件设备（在这里指NPU芯片）直接分配给某个虚拟机或容器。这样，虚拟机内的应用可以绕过虚拟化层的常规调度，获得几乎原生的硬件访问权限。
    
- **性能优势：**  
    通过直接访问，应用能够充分利用NPU的计算能力，避免了共享设备时因中间层调度带来的额外性能开销。
    

### 2. 为什么要将整个NPU芯片分配给单个实例？

- **资源独占：**  
    独占整个NPU芯片可以确保该应用实例能够获得芯片的所有资源，而不必与其他实例竞争。这样可以避免因资源共享而导致的性能瓶颈。
    
- **安全与隔离：**  
    在多租户云环境中，隔离性非常关键。将整个芯片分配给单个实例能够增强各个实例之间的隔离，降低相互干扰或数据泄露的风险。
    
- **简化资源管理：**  
    独占方式使得资源调度和管理更加简单，系统可以为高负载的机器学习任务提供更稳定和可预测的性能，而不必担心其他任务占用部分硬件资源。
    

### 3. 实际意义

- **提高计算效率：**  
    直接利用整个NPU芯片可以实现低延迟和高吞吐量，这对于计算密集型的机器学习任务来说尤为重要。
    
- **设计权衡：**  
    虽然这种方法可能在某些情况下导致NPU资源的利用率不够充分，但对于要求高性能和严格隔离的应用来说，其带来的性能和安全优势通常更为重要。
    

总之，通过PCIe直通技术将整个NPU芯片分配给单个ML应用实例，能够提供更高的性能、更好的资源隔离以及更简单的资源管理，适用于对计算能力和安全性有较高要求的云平台应用。

However, this disables resource sharing and causes severe resource underutilization of NPUs. Many DNN workloads have diverse demands on the number of MEs and VEs. As a result, the one-size-fits-all approach is much less attractive for cloud platforms.

To address the utilization challenge and ease the resource management for cloud platforms to accommodate diverse work-load demands, it is desirable to virtualize hardware devices and enable resource sharing among multiple tenants. Unfortunately, modern cloud platforms have very limited virtualization support for NPUs across the software and hardware stack.

缺乏对 NPU 的系统抽象支持。与多核处理器的系统虚拟化 [3]、[10] 不同，NPU 具有独特的异构计算资源（即 ME 和 VE）。为了避免这种复杂性，如今的云平台将同质的 NPU 核心暴露给用户 VM。然而，现有的 NPU 核心级别的抽象太粗粒度，因为用户工作负载可能具有不同的资源需求。我们需要一个灵活的系统抽象，允许用户按照即用即付模式指定 ME/VE 资源 [48]。这种抽象将简化云平台的 NPU 管理，包括 NPU 资源（取消）分配、资源映射和调度。先前的研究调查了 FPGA [6]、[33]、[34]、[63]、[64] 和 GPU [26]、[55] 的系统虚拟化。然而，由于它们针对的是不同的架构，因此它们不能直接应用于 NPU。

缺乏对 NPU 虚拟化的架构支持。先前的研究在任务级别启用了 NPU 设备的分时，并支持优先级任务的抢占 [12]、[13]。然而，由于缺乏对多租户工作负载并发执行的支持，共享 NPU 板上的粗粒度分时仍然遭受严重的资源利用不足。现有的 NPU 共享方法要么牺牲隔离，要么遭受高抢占开销 [16]。V10 [59] 启用了多个 DNN 工作负载之间的 NPU 共享。然而，它仍然基于分时机制，并且受到多租户 ML 实例之间运算符干扰的影响，导致性能隔离不佳。随着我们走向细粒度的 NPU 虚拟化，我们需要架构支持来实现改进的性能隔离和 NPU 利用率。

缺乏对虚拟化 NPU 的 ISA 支持。为了简化硬件设计，NPU 通常采用 VLIW 样式的 ISA，而 ML 编译器明确利用了计算单元的并行性 [5]、[28]、[32]。然而，这需要在编译阶段明确指定计算单元的数量，并且该数量在运行时无法更改。在这种情况下，VLIW ISA 不必要地耦合了计算单元（即 ME）的控制流。即使共享 NPU 的一些计算单元可用，它们也不能被活动工作负载使用（重新编译 DNN 程序除外）。这是由动态调度和 VLIW ISA 之间的根本争斗造成的。由于共置的 ML 实例在运行时对计算单元有各种需求，这种限制不可避免地会导致 NPU 利用率不足或性能干扰。我们需要重新思考 NPU ISA 的设计，以促进虚拟化 NPU 的动态资源调度。

理想情况下，我们希望虚拟化 NPU，以实现灵活、细粒度的资源共享和调度，从而提高 NPU 利用率和性能隔离。我们提出了 Neu10，一种用于 NPU 的硬件辅助系统虚拟化框架。

我们的贡献。我们首先开发一个简单而灵活的 vNPU 抽象。我们使用 vNPU 为每个 ML 实例创建一个虚拟化的 NPU 设备。对于每个 vNPU，用户可以按需指定不同类型的计算单元 (ME/VE) 的数量，或者遵循云计算中的即用即付模式。我们提出了一种新的资源分配机制，该机制可以根据使用 ML 编译器的分析，为不同的 ML 工作负载决定优化的 vNPU 配置。由于不同的 ML 服务具有不同的 ME/VE 需求（参见 §II），这种抽象可以实现细粒度的资源分配，这对最终用户和云平台运营商都有好处。

Neu10 可以根据 ML 服务的服务级别目标 (SLO)，以不同的方式将 vNPU 映射到 NPU 核心的物理计算单元。为了在确保性能隔离的同时最大限度地提高 NPU 利用率，Neu10 通过资源收集实现了细粒度的空间共享。它还通过在多个 vNPU 之间临时共享 ME/VE 来实现 NPU 核心的超额认购。因此，闲置的计算单元可以被共置工作负载适时利用。

为了促进共置 vNPU 的动态调度，Neu10 通过将 VLIW 指令重组为独立的微张量运算符（§III 中的 µTOps）来扩展 VLIW 样式 ISA。Neu10 引入了在共享物理 NPU 核心上对 µTOps 进行细粒度动态调度所需的架构逻辑。它允许一个 vNPU 从共置 vNPU 中获取 ME/VE 的可用计算周期，而不会造成太多干扰。这在传统的 VLIW 样式 ISA 中是不可能的，因为它们严格耦合了（静态）分配的计算单元的控制流。我们的新架构支持使 Neu10 能够提供跨软件（即 vNPU 抽象）和硬件（即细粒度 µTOp 调度）堆栈的 NPU 资源分配和调度的灵活性。Neu10 需要对 NPU 芯片（0.04% 的芯片面积成本）以及 ML 编译器进行最低限度的修改。

我们根据典型的 TPU 架构，使用生产级 NPU 模拟器实现了 Neu10。我们在真实的 Google TPU 上运行 MLPerf 基准测试 [46] 和 TPU 参考模型 [22] 时收集 ML 服务的踪迹。我们对多租户 ML 实例进行的实验表明，与最先进的 NPU 共享方法相比，Neu10 可以将 ML 推理服务的吞吐量提高多达 1.4 倍，并将尾部延迟降低多达 4.6 倍，同时将 NPU 利用率平均提高 1.2 倍。我们总结了 Neu10 的贡献如下：
- 我们对真实 NPU 硬件上的 DNN 推理工作负载进行了深入研究，并研究了系统和硬件堆栈中的 NPU 虚拟化挑战
- We propose a new system abstraction named vNPU for enabling fine-grained virtualization of the heterogenerous compute units in NPU cores.
- We present a new NPU resource allocation scheme and enable flexible vNPU-to-pNPU mappings.
- We extend the VLIW-style ISAs and NPU architecture for enabling fine-grained dynamic scheduling of virtualized NPUs for multi-tenant ML services.
- We evaluate the efficiency and flexibility of our NPU virtualization framework with real-world DNN traces.