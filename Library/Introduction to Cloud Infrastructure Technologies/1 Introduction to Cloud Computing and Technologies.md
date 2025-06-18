从历史上看，“云”一词曾被用作互联网的隐喻，而云形图形则被用于计算机网络图中，以表示互联网。从 20 世纪 60 年代初开始，大型计算机使多个用户能够在共享的内部计算资源上同时运行他们的程序。三十年后，虚拟专用网络 (VNP) 成为一种流行的手段，不仅可以互连分布式计算资源，还可以让远距离的用户访问这些计算资源。

云计算是一种计算模型，它允许远程分配第三方拥有的计算资源，以供最终用户使用。

云计算是一种模型，它允许通过无处不在、便捷、按需的网络访问可配置的计算资源共享池（例如网络、服务器、存储、应用程序和服务），这些资源可以通过最少的管理工作或服务提供商交互快速配置和发布”。

云计算提供商提供各种基于工具构建的服务，这些工具可自动执行基本资源的配置和发布。根据 NIST 的定义，这些服务模型大多属于以下类别之一：
- Infrastructure as a Service (IaaS)
- Platform as a Service (PaaS)
- Software as a Service (SaaS).

然而，近年来，已经引入了各种附加服务模型来描述远程计算服务客户的需求和要求：
- Analytics as a Service (AnaaS)
- API as a Service (AaaS)
- Big Data as a Service (BDaaS)
- Business Process as a Service (BPaaS)
- Code as a Service (CaaS)
- Communications Platform as a Service (CPaaS)
- Desktop as a Service (DaaS)
- Database as a Service (DBaaS)
- Function as a Service (FaaS)
- Monitoring as a Service (MaaS)
- Anything as a Service (XaaS).

在整个课程中，我们将探索服务和部署模型以及关键特性。云服务提供商（云计算资源的所有者）以控制台和仪表板的形式提供 Web 界面，旨在提升平台上的用户体验，并简化所需服务堆栈的构建、配置和监控。云服务模型的另一个优势是云提供商采用的按需付费模式，用户只需为在规定时间段内使用的云资源付费。这种将计算资源视为实用工具的方法，消除了在硬件、软件和专业人员方面高额的前期投资。


# Key Characteristics of Cloud Computing
Most beneficial characteristics of cloud computing are:
速度与敏捷性
只需点击几下即可配置云资源，节省时间并提高敏捷性。
成本
通过降低设置基础设施的前期成本，用户可以专注于支持业务流程的应用程序。云提供商的另一个优势是成本估算器，它可以帮助用户更好地规划资源。
轻松访问资源
只要连接到云服务提供商，用户就可以从任何地点和设备访问云服务。
维护
所有用于保持资源正常运行和更新的日常维护工作都由提供商承担，用户无需再操心。
多租户
多个用户共享同一个资源池，这有助于降低每个用户的成本。
可靠性
资源可以托管在冗余的数据中心位置，以提供更高的可靠性、弹性和高可用性。
可扩展性和弹性
自助服务配置与功能相结合，使云资源能够动态地向上扩展（规模扩大）和向外扩展（复制），以满足客户需求或保持最佳的资源利用率。向下扩展（规模缩小）和向内扩展也实现了自动化。
安全性
分布式设备上存储的数据和运行的应用程序增加了安全需求的复杂性，而云服务提供商的专业资源可以轻松解决这一问题。


# Cloud Deployment Models
Primarily, cloud is deployed under the following models:
- Private Cloud
它仅由一个组织指定和运营。它可以托管在内部或外部，并由内部团队或第三方管理。可以使用 OpenStack 等软件堆栈构建私有云。

- Public Cloud
- Hybrid Cloud
Public and private clouds can be bound together to offer a hybrid cloud. Such hybrid clouds are beneficial for:
1. Storage of sensitive information on the private cloud, while offering public services based on that information from a public cloud.
2. Meeting temporary resources needs during peak or times of high demand by scaling to the cloud, known as bursting, when such temporary needs of additional computing resources cannot be met by the private cloud.

Additional cloud deployment models may include: 
- _Community Cloud_  
    Formed by multiple organizations sharing a common cloud infrastructure.
- _Distributed Cloud_  
    Formed by distributed systems connected by a single network.
- _Multicloud_  
    One organization uses multiple public cloud providers to run its workload, typically to avoid vendor lock-in.
- _Poly Cloud_  
    One organization uses multiple public cloud providers to leverage specific services from each provider.

# Virtualization
为了促进云部署模型和实现云计算的特性，云服务提供商和自管理云部署工具利用专门的软件来实现硬件虚拟化。

在计算中，创建物理计算资源的虚拟版本的能力（包括虚拟计算机硬件平台、操作系统、虚拟存储设备和虚拟计算资源）称为虚拟化。

虚拟化可以在不同的硬件和软件层面实现，例如中央处理器 (CPU)、存储（磁盘）、内存 (RAM)、文件系统等。在本章中，我们将探索一些工具，这些工具允许我们通过虚拟化支持在其上安装客户操作系统 (OS) 的基本硬件组件来创建虚拟机 (VM)。虚拟机是硬件构建的计算机器的软件等价物，它代表一组独立的虚拟资源，其行为类似于实际的物理系统。

虚拟机是在专用虚拟化软件（即虚拟机管理程序）的帮助下创建的，该软件在主机上运行。虚拟机管理程序是一种能够创建多个隔离的虚拟操作环境的软件，每个虚拟操作环境由虚拟化资源组成，然后可供客户系统使用。虚拟机管理程序主要分为两类：

_Type-1 hypervisor_ (native or bare-metal) runs directly on top of a physical host machine's hardware without the need for a host OS. Typically, they are found in enterprise settings. Examples of type-1 hypervisors:

- AWS Nitro
- IBM z/VM
- Microsoft Hyper-V 
- Nutanix AHV 
- Oracle VM Server for SPARC
- Oracle VM Server for x86
- Red Hat Virtualization
- VMware ESXi
- Xen.

_Type-2 hypervisor_ (hosted) runs on top of the host's OS. Typically, they are for end-users, but they may be found in enterprise settings as well. Examples of type-2 hypervisors:

- Parallels Desktop for Mac
- VirtualBox
- VMware Player
- VMware Workstation.

但也有一些例外——同时符合两种类别的虚拟机管理程序，它们可以冗余地列在类型 1 和类型 2 的虚拟机管理程序下。它们是同时充当类型 1 和类型 2 虚拟机管理程序的 Linux 内核模块。例如：
- KVM
- bhyve.

虚拟机管理程序 (Hypervisor) 支持计算机硬件（例如 CPU、磁盘、网络、内存等）的虚拟化，并允许在其上安装客户虚拟机 (VM)。我们可以在单个虚拟机管理程序上创建多个运行不同操作系统的客户虚拟机。例如，我们可以使用原生 Linux 计算机作为主机。设置好 Type-2 虚拟机管理程序后，我们可以创建多个运行不同操作系统（通常是 Linux 和 Windows）的客户虚拟机。

大多数现代 CPU 都支持硬件虚拟化，该功能允许虚拟机管理程序虚拟化主机系统的物理硬件，从而以安全高效的方式与多个客户系统共享主机系统的处理资源。此外，大多数现代 CPU 还支持嵌套虚拟化，该功能允许在另一个虚拟机内创建虚拟机。

虽然虚拟化是将物理硬件转换为虚拟化软件资源以进行虚拟机配置的关键方法，但也可以借助模拟器来创建虚拟系统。更具体地说，像 QEMU 这样的工具支持在系统和用户空间层面进行的软件模拟，允许任何操作系统在任何架构上运行，或允许程序在原本不受支持的系统上运行。用户还可以利用 QEMU 的灵活性，将其与 KVM 等虚拟机管理程序结合使用来配置虚拟机。

接下来，让我们探索几种借助虚拟机管理程序创建虚拟机的方法。

By the end of this chapter, you should be able to:

- Explain the concept of virtualization.
- Explain the role of a hypervisor in the creation of Virtual Machines.
- List hypervisor types. 
- Create and configure Virtual Machines with hypervisors, such as KVM and VirtualBox.
- Explore methods to automate the creation of VMs with Vagrant.

# Introduction to KVM
根据 linux-kvm.org 的介绍，

“KVM（基于内核的虚拟机）是针对 x86 硬件上的 Linux 的完整虚拟化解决方案”。

KVM 是 Linux 内核的一个可加载虚拟化模块，它将内核转换为能够管理客户虚拟机的虚拟机管理程序 (hypervisor)。大多数现代处理器支持的特定硬件虚拟化扩展（例如 Intel VT 和 AMD-V）必须可用并启用，处理器才能支持 KVM。虽然 KVM 最初是为 x86 硬件设计的，但也已移植到 FreeBSD、S/390、PowerPC、IA-64 和 ARM 平台。
![[Pasted image 20250522215342.png]]
KVM is an open source software that provides hardware-assisted virtualization to support [various guest OSes](http://www.linux-kvm.org/page/Guest_Support_Status), such as Linux distributions, Windows, Solaris, etc.

KVM enables device abstraction of network interfaces, disk, but not the processor. Instead, it exposes the **/dev/kvm** interface that can be used by an external user space host for emulation. [KubeVirt](https://kubevirt.io/), [QEMU](https://www.qemu.org/), and [virt-manager](https://virt-manager.org/) are examples of user space tools for [KVM Virtual Machine management](https://www.linux-kvm.org/page/Management_Tools).

KVM 支持嵌套客户机 (nested guest)，此功能允许虚拟机 (VM) 在虚拟机 (VM) 中运行。它还支持热插拔设备，例如 CPU 和 PCI 设备。对于系统上可能不可用的额外虚拟化资源的分配，KVM 也可以实现过量使用。为了实现虚拟机的过量使用，KVM 会动态地与另一个未使用所需资源类型的客户机 (guest) 交换资源。为了实现虚拟机的过量使用，KVM 会动态地与另一个未使用所需资源类型的客户机 (guest) 交换资源。