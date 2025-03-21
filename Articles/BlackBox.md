# Abstract
Containers rely on the operating system to enforce their security guatantees. This poses a significant security risk as large operating system codebases contain many vulnerabilities. We have created BlackBox, a new container architecture that provides fine-grain protection of application data confidentiality and integrity without trusting the operating system. ==BlackBox introduces a container security monitor, a small trusted computing base that creates protected physical address spaces (PPASes) for each container such that there is no direct information flow from container to OS or other container PPASes.== Indirect information flow can only happen through the monitor, which only copies data between container PPASes and the operating system as system call arguments, encrypting data as needed to protect interprocess communication through the operating system. Containerized applications do not need to be modified, can still make use of operating system services via system calls, yet their CPU and memory state are isolated and protected from other containers and the operating system. We have implemented BlackBox by leveraging Arm hardware virtualization support, using nested paging to enforce PPASes.

# Intro
Popular container mechanisms such as Linux containers rely on a commodity operating system (OS) to enforce their security guarantees. 

Modern systems incorporate hardware security mechanisms to protect application from an untrusted OS, such aas Inter Software Guard Extensions (SGX), but they require rewriting applications and may impose high overhead to use OS services. Some approaches have built on these mechanisms to protect unmodified applications or containers. Unfortunately, they suffer from high overhead, incomplete and limited functionality, and massively increase the trusted computing base (TCB) through a library OS or runtime system, potentially trading one large vulnerable TCB for another.

作为替代方案，虚拟机管理程序已增强了其他机制，以保护应用程序免受不受信任的操作系统的攻击 [11,12,27,35,67]。这会导致基于虚拟机管理程序的虚拟化的性能开销，而容器的设计初衷就是为了避免这种情况。这些系统的 TCB 非常大，在某些情况下包括额外的商品主机操作系统，从而提供了额外的漏洞，可利用这些漏洞来危害应用程序。从理论上讲，这些方法可以应用于具有较小 TCB 的微型虚拟机管理程序 [10,61]。不幸的是，微型虚拟机管理程序仍然继承了基于虚拟机管理程序的虚拟化的复杂性，包括虚拟化和管理硬件资源。TCB 的减少是通过大大减少的功能集和有限的硬件支持来实现的，这使得它们在实践中部署起来很困难。

To address this problem, we have created BlackBox, a new container architecture that provides fine-grain protection of application data confidentiality and intergrity without the need to trust the OS. BlackBox introduces a _container security moniter (CSM)_, a new mechanism that leverages existing hardware feature to enforce container security guarantees in a small trusted computing base (TCB) in lieu of the OS. The monitor creates protected physical address spaces (PPASes) for each container to enforce physical address spaces (PPASes) for each container to enforce physical memory access controls, but provides no virtualization of hardware resources. 监视器为每个容器创建受保护的物理地址空间 (PPAS)，以强制执行物理内存访问控制，但不提供硬件资源的虚拟化。映射到容器 PPAS 的物理内存在 PPAS 之外无法访问，从而在容器和操作系统之间提供物理内存隔离。由于物理内存中的容器私有数据仅驻留在其自己的 PPAS 中的页面上，因此其机密性和完整性受到保护，不受操作系统和其他容器的影响。

CSM 重新利用现有的硬件虚拟化支持，以更高的权限级别运行并创建 PPAS，但其本身并不是虚拟机管理程序，也不会虚拟化硬件。相反，操作系统继续直接访问设备并负责分配资源。这使得 CSM 既简约又简单，同时保持高性能。通过直接支持容器而无需虚拟化，无需在安全执行环境中运行额外的客户操作系统或复杂的运行时，从而最大限度地减少了容器本身内的 TCB。

在 BlackBox 容器中运行的应用程序无需修改，可以通过系统调用使用操作系统服务，同时还可以保护其数据不受操作系统的影响。监视器介入容器和操作系统之间的所有转换，清除 CPU 寄存器中的容器私有数据并根据需要切换 PPAS。内存中的任何容器数据唯一可供操作系统使用的时间是作为系统调用参数，只有监视器本身可以通过在容器 PPAS 和操作系统之间复制参数来提供。监视器了解系统调用语义，并在将系统调用参数传递给操作系统之前根据需要对其进行加密，例如用于进程之间的进程间通信，保护系统调用参数中的容器私有数据不受操作系统的影响。鉴于端到端加密在 I/O 安全方面的使用越来越多[55]，部分原因是斯诺登泄密事件[36]，监视器依靠应用程序加密自己的 I/O 数据来简化其设计。一旦系统调用完成并且允许进程返回其容器之前，监视器会检查 CPU 状态以验证进程，然后再将 CPU 切换回使用容器的 PPAS。

除了确保容器的 CPU 和内存状态在容器外部不可访问外，BlackBox 还可以防止在容器内运行的恶意代码。只有经过签名和加密的受信任二进制文件才能在 BlackBox 容器中运行。监视器需要解密二进制文件，因此它们只能在监视器监督的 BlackBox 容器内运行。监视器在二进制文件运行之前对其进行身份验证，因此不受信任的二进制文件无法在 BlackBox 容器中运行。它还可以防止与内存相关的 Iago 攻击，这种攻击会恶意操纵虚拟和物理内存映射，通过阻止可能覆盖进程堆栈的虚拟或物理内存分配，可能导致容器中进程的任意代码执行。

鉴于 Arm 在个人计算机和云计算基础设施中的应用日益广泛，以及在移动和嵌入式系统上的主导地位，我们在 Arm 硬件上实现了 BlackBox。我们利用 Arm 硬件虚拟化支持，重新利用 Arm 的 EL2 特权级别和嵌套分页（最初设计用于运行虚拟机管理程序），以强制分离 PPAS。与运行虚拟机管理程序的 x86 根操作不同，Arm EL2 具有自己的硬件系统状态。这最大限度地降低了在调用和返回系统调用时捕获到在 EL2 中运行的监视器的成本，因为系统状态不必在每个陷阱上保存和恢复。我们表明，BlackBox 只需对 Linux 内核进行适度修改即可支持广泛使用的 Linux 容器，并从操作系统继承了对各种 Arm 硬件的支持。该实现的 TCB 不到 5K 行代码，外加一个经过验证的加密库，比商用操作系统和虚拟机管理程序少几个数量级。由于尺寸如此小，开发人员可以更轻松地维护 CSM 并确保其正确性，甚至比仅维护虚拟机管理程序的核心虚拟化功能更容易。我们表明，与传统的虚拟机管理程序和容器架构相比，BlackBox 可以提供更细粒度和更强大的安全保障，而实际应用程序工作负载的性能开销仅为中等。

# Threat Model and Assumptions
我们的威胁模型主要关注操作系统漏洞，这些漏洞可能被利用来破坏容器私有数据的机密性或完整性。范围内的攻击包括破坏操作系统或任何其他软件以读取或修改私有容器内存或寄存器状态，包括通过控制支持 DMA 的设备，或通过内存重新映射和别名攻击。我们假设容器不会故意或无意地主动泄露自己的私有数据，但来自其他受损容器的攻击（包括机密性和完整性攻击）都在范围内。受损操作系统的可用性攻击超出了范围。物理或旁道攻击 [5、32、43、52、68、69] 超出了本文的范围。BlackBox 中发生旁道攻击的机会比在较低级别隔离的系统（例如虚拟机）中更大。 BlackBox 的信任边界是操作系统的系统调用 API，这使得对手能够看到操作系统交互的一些细节，例如大小和偏移量。

我们假设安全密钥存储可用，例如由可信平台模块 (TPM) 提供的 [31]。我们假设硬件没有错误，系统最初是良性的，允许在系统受到损害之前安全地存储签名和密钥。我们假设容器使用端到端加密通道来保护其 I/O 数据 [21、37、55]。我们假设 CSM 没有任何漏洞，因此可以信任；正式验证其代码库是未来的工作。我们假设对任何加密容器数据执行暴力攻击在计算上都是不可行的，并且假设任何加密通信协议都旨在防御重放攻击。

# Design
BlackBox 将传统 Linux 容器置于飞地中，以保护容器数据的机密性和完整性。如果 BlackBox 保护容器免受操作系统的影响，我们称该容器为飞地容器。从应用程序的角度来看，使用飞地容器与使用传统容器没有太大区别。应用程序无需修改即可使用飞地容器，并且可以通过系统调用使用操作系统服务。容器管理解决方案 [48,49]（如 Docker [20]）可用于管理飞地容器。BlackBox 旨在支持商用操作系统，尽管使用其飞地机制需要对操作系统进行微小修改，这与通常需要对操作系统进行修改才能利用新硬件功能的方式非常相似。但是，BlackBox 不信任操作系统，运行飞地容器的受损操作系统无法破坏其数据的机密性和完整性。

![[Pasted image 20250318153732.png]]

BlackBox 引入了容器安全监视器 (CSM)，如图 1 所示，它充当其 TCB。CSM 的唯一目的是保护正在使用的容器数据的机密性和完整性。它通过执行两个主要功能（访问控制和验证操作系统操作）来实现这一点。其狭窄的目的和功能使 CSM 能够保持小巧和简单，避免许多其他受信任的系统软件组件的复杂性。例如，与虚拟机管理程序不同，CSM 不会虚拟化或管理硬件资源。它不维护虚拟硬件（例如虚拟 CPU 或设备），从而避免了模拟 CPU 指令、中断或设备的需要。相反，中断直接传递给操作系统，设备由操作系统的现有驱动程序直接管理。它也不执行 CPU 调度或内存分配，因此不提供可用性保证。 CSM 可以保持较小规模，因为它假定操作系统能够感知 CSM，并且依赖操作系统实现复杂的功能，例如引导、CPU 调度、内存管理、文件系统以及中断和设备管理。

为了封装容器，CSM 引入了受保护的物理地址空间 (PPAS) 的概念，即一组独立的物理内存页面，只有 PPAS 的指定所有者和 CSM 才能访问。每个物理内存页面最多映射到一个 PPAS。CSM 使用此机制通过为每个封装容器分配一个单独的 PPAS 来提供内存访问控制，从而将每个容器的物理内存与操作系统和任何其他容器隔离开来。操作系统确定为每个 PPAS 分配了哪些内存，但无法访问 PPAS 的内存内容。同样，容器无法访问它不拥有的 PPAS。未分配给 PPAS 或 CSM 的内存已分配给操作系统并可供操作系统访问。CSM 本身可以访问任何内存，包括分配给 PPAS 的内存。在 PPAS 内，访问内存的地址与机器上的物理地址相同；物理内存不能重新映射到 PPAS 中的其他地址。例如，如果将物理内存的页码 5 分配给 PPAS，它将在 PPAS 内作为页码 5 进行访问。内存中的容器私有数据仅驻留在映射到其自己的 PPAS 的页面上，因此其机密性和完整性受到操作系统和其他容器的保护。第 4 节描述了 BlackBox 如何使用嵌套页表来强制执行 PPAS。

CSM 介入容器和操作系统之间的所有转换，即系统调用、中断和异常，以确保进程和线程（我们统称为任务）在容器上下文中执行时只能访问其所属容器的 PPAS。CSM 确保当任务陷入操作系统并切换到运行操作系统内核代码时，该任务不再有权访问容器的 PPAS。否则，操作系统可能会导致任务访问容器的私有数据，从而损害其机密性或完整性。CSM 维护一个封闭任务数组，该数组包含封闭容器中运行的所有任务的信息。进入操作系统时，CSM 检查调用任务是否在封闭容器中，在这种情况下，它会将 CPU 寄存器和陷阱原因保存到封闭任务数组中，切换出容器的 PPAS，并清除操作系统不需要的任何 CPU 寄存器。退出操作系统时，CSM 检查隔离任务数组，以确定正在运行的任务是否属于隔离容器，在这种情况下，它会验证当前 CPU 上下文（即堆栈指针和页表基址寄存器）是否与隔离任务数组中保存的相应任务相匹配。如果匹配，CSM 会切换到相应容器的 PPAS，以便任务可以访问其隔离 CPU 和内存状态。因此，CPU 寄存器或内存中的容器私有数据无法被操作系统访问。

为了支持传统上需要访问任务的 CPU 状态和内存的 OS 功能，CSM 为 OS 提供了一个应用程序二进制接口 (ABI)，以便 OS 从 CSM 请求服务。CSM ABI 如表 1 所示。例如，create_enclave 和 destroy_enclave 分别由 OS 调用，以响应容器运行时（例如 runC [29]）对 enclave 和 unenclave 容器的请求。对于需要动态分配内存的 CSM 调用，OS 必须分配并传入足够大的连续内存区域的物理地址以执行相应的操作。否则，调用将失败并返回所需的内存量，以便 OS 可以使用所需的分配再次进行调用。例如，create_enclave 要求 OS 分配内存以用于 enclave 容器的元数据。成功后，分配的内存将分配给 CSM，并且操作系统将无法再访问它，直到调用 destroy_enclave，此时内存将再次分配回操作系统。

TODO



### BlackBox容器安全监控系统关键总结

#### 一、核心创新与架构设计

1. **容器安全监控器（CSM）**
    - 作为可信计算基（TCB），仅约5K行代码，显著小于传统OS/虚拟机监控器。
    - 不虚拟化硬件资源，仅通过**受保护物理地址空间（PPAS）**实现内存隔离。
    - 利用ARM EL2特权级和嵌套分页（Nested Paging）强制隔离容器内存。
2. **物理地址空间保护（PPAS）**
    - 每个容器独占一个PPAS，物理内存页仅允许所属容器和CSM访问。
    - 通过硬件机制阻止操作系统和其他容器直接访问受保护内存。
    - 内存地址在PPAS内与物理地址一致，避免重映射攻击。
3. **系统调用与上下文切换**
    - 拦截所有容器与OS的交互（系统调用、中断、异常），清除寄存器残留数据。
    - 仅在系统调用参数传递时复制数据，并通过加密保护跨容器通信。
    - 维护"enclaved task array"记录任务状态，验证上下文完整性。

#### 二、安全机制与威胁防护

1. **数据保护层级**
    - **内存隔离**：物理内存按PPAS划分，阻止越界访问。
    - **CPU状态保护**：切换上下文时清除敏感寄存器数据。
    - **加密传输**：系统调用参数加密，依赖应用层端到端加密保障I/O安全。
2. **攻击防御能力**
    - 防范操作系统漏洞利用（如内存读取/篡改、DMA攻击）。
    - 阻止恶意容器间的横向渗透（信息流仅通过CSM可控复制）。
    - 通过签名验证和内存分配限制抵御Iago攻击（堆栈覆盖攻击）。
3. **可信启动与密钥管理**
    - 依赖UEFI安全启动验证CSM和OS镜像签名。
    - 密钥存储基于TPM等硬件安全模块，防止离线破解。

#### 三、实现与性能

1. **ARM硬件支持**
    - 利用EL2特权级的独立系统状态，减少上下文切换开销。
    - 嵌套分页实现PPAS隔离，无需修改容器化应用。
2. **兼容性与扩展性**
    - 支持主流Linux容器（如Docker），仅需少量内核修改。
    - 继承OS对硬件驱动的支持，避免重复开发设备管理模块。
3. **性能表现**
    - 系统调用延迟优化：EL2硬件状态保留减少寄存器保存/恢复操作。
    - 实测应用负载下性能损失有限（具体数据未提供但强调"modest overhead"）。
![[Pasted image 20250318153012.png]]

#### 五、局限性及未来方向

1. **未覆盖的攻击面**
    - 侧信道攻击（如缓存计时攻击）未被防御。
    - 假设硬件无漏洞，实际部署需结合其他防护措施。
2. **功能限制**
    - 不提供可用性保障（OS仍可拒绝服务）。
    - 依赖应用层实现I/O加密，未内置全链路加密。
3. **验证与扩展**
    - CSM代码尚未形式化验证（论文提及为未来工作）。
    - 当前仅支持ARM架构，x86移植需评估可行性。

> **总结**：BlackBox通过极简的TCB设计和硬件辅助隔离机制，在保持容器轻量级特性的同时，实现了接近虚拟机的安全强度。其创新点在于重新定义容器与OS的信任边界，为云原生场景下对抗不可信操作系统提供了新思路。

