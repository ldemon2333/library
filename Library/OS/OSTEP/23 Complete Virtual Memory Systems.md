How to build a complete VM system?
What features are needed  to realize a complete virtual memory system? How do they improve performance, increase security, or otherwise improve the system?

We'll do this by covering two system. VAX/VMS and Linux.

# 23.1 VAX/VMS Virtual Memory
The OS for the system was known as VAX/VMS, one of whose primary architects was Dave Cutler, who later led the effort to develop Microsoft's Windows NT.

## Memory Management Hardware
The VAX-11 provided a 32-bit virtual address space per process, divided into 512 byte pages. Thus, a virtual address consisted of a 23-bit VPN and a 9-bit offset. Further, the upper two bits of the VPN were used to differentiate which segment the page resided within; thus, the system was a hybrid of paging and segmentation.

![[Pasted image 20241127145444.png]]

The lower-half of the address space was known as "process space" and is unique to each process. In the first half of process space (known as P0). The upper-half of the address space is known as system space (S), although only half of it is used. Protected OS code and data reside here, and the OS is in this way shared across processes.

## A Real Address Space
One neat aspect of studying VMS is that we can see how a real address space is constructed (Figure 23.1). Thus far, we have assumed a simple address space of just user code, user data, and user heap, but as we can see above, a real address space is notably more complex.

For example, the code segment never begins at page 0. This page, instead, is marked inaccessible, in order to provide some support for detecting **null-pointer** access.

# 23.2 The Linux Virtual Memory System
Linux for Intel x86

## The Linux Address Space
![[Pasted image 20241127153216.png]]

One slightly interesting aspect of Linux is that it contains two types of kernel virtual addresses. The first are known as **kernel logical addresses**. This is what you would consider the normal virtual address space; to get more memory of this type, kernel code merely needs to call **kmalloc**. Most kernel data structures live here, such as page tables, per-process kernel stacks, and so forth. Unlike most other memory in the system, kernel logical memory cannnot be swapped to disk.

内核逻辑地址最有趣的方面是它们与物理内存的连接。具体来说，内核逻辑地址和物理内存的第一部分之间存在直接映射。因此，内核逻辑地址 0xC0000000 转换为物理地址 0x00000000，0xC0000FFF 转换为 0x00000FFF，依此类推。这种直接映射有两个含义。首先，在内核逻辑地址和物理地址之间来回转换很简单；因此，这些地址通常被视为确实是物理地址。
第二，如果一块内存在内核逻辑地址空间中是连续的，那么它在物理内存中也是连续的。这使得在内核地址空间的这一部分分配的内存适用于需要连续物理内存才能正常工作的操作，例如通过直接内存访问 (DMA) 进行的 I/O 传输。

The other type of kernel address is a **kernel virtual address**. To get memory of this type, kernel code calls a different allocator, **vmalloc**, which returns a pointer to a virtually contiguous region of the desired size. 与内核逻辑内存不同，内核虚拟内存通常不连续；每个内核虚拟页面可能映射到不连续的物理页面（因此不适合 DMA）。但是，这种内存因此更容易分配，因此可用于大型缓冲区，在这些缓冲区中很难找到连续的大块物理内存。

## Page Table Structure
Moving to a 64-bit address affects page table structure in x86 in the expected manner. 
![[Pasted image 20241127155117.png]]
As you can see in the picture, the top 16 bits of a virtual address are unused, 低 12 位是（4KB）是页大小。中间是 4 级页表

## Large Page Support
Recent designs support 2-MB and even 1-GB pages in hardware. Thus, over time, Linux 使用 huge pages.

## The Page Cache
为了降低访问持久存储的成本（本书第三部分的重点），大多数系统使用积极的缓存子系统将热门数据项保存在内存中。在这方面，Linux 与传统操作系统没有什么不同。

Linux 页面缓存是统一的，将页面保存在内存中的三个主要来源：内存映射文件、来自设备的文件数据和元数据（通常通过将 read() 和 write() 调用定向到文件系统来访问），以及组成每个进程的堆和堆栈页面（有时称为匿名内存，因为其下方没有命名文件，而是交换空间）。这些实体保存在页面缓存哈希表中，以便在需要所述数据时快速查找

页面缓存会跟踪条目是干净的（已读取但未更新）还是脏的（即已修改）。脏数据会定期由后台线程（称为 pdflush）写入后备存储（即，写入特定文件以存储文件数据，或交换空间以存储匿名区域），从而确保修改后的数据最终会写回到持久存储。此后台活动要么在一定时间段后发生，要么在太多页面被视为脏时发生（两者均为可配置参数）。

在某些情况下，系统内存不足，Linux 必须决定将哪些页面踢出内存以释放空间。为此，Linux 使用了一种经过修改的 2Q 替换形式，我们在此处对其进行了描述。

基本思想很简单：标准 LRU 替换是有效的，但某些常见的访问模式可能会破坏这种替换。例如，如果一个进程反复访问一个大文件（尤其是接近内存大小或更大的文件），LRU 会将其他所有文件踢出内存。更糟糕的是：将此文件的部分保留在内存中是没有用的，因为它们在被踢出内存之前从未被重新引用过。

Linux 版本的 2Q 替换算法通过保留两个列表并在它们之间划分内存来解决此问题。当第一次访问时，页面被放置在一个队列中（在原始论文中称为 A1，但在 Linux 中称为非活动列表）；当它被重新引用时，页面被提升到另一个队列（在原始论文中称为 Aq，但在 Linux 中称为活动列表）。当需要进行替换时，替换候选者将从非活动列表中取出。 Linux 还会定期将页面从活动列表底部移动到非活动列表，使活动列表保持为总页面缓存大小的三分之二左右。Linux 理想情况下会以完美的 LRU 顺序管理这些列表，但如前几章所述，这样做成本很高。因此，与许多操作系统一样，使用 LRU 的近似值（类似于时钟替换）。

## Security And Buffer Overflows
现代虚拟机系统（Linux、Solaris 或 BSD 变体之一）与古老虚拟机系统（VAX/VMS）之间最大的区别可能在于现代时代对安全性的重视。保护一直是操作系统的严重关注点，但随着机器之间的互联程度比以往任何时候都高，开发人员实施了各种防御措施来阻止那些狡猾的黑客控制系统也就不足为奇了。

缓冲区溢出攻击是主要威胁之一，这种攻击可用于对付普通用户程序，甚至内核本身。这些攻击的目的是在目标系统中寻找漏洞，让攻击者将任意数据注入目标的地址空间。此类漏洞有时是因为开发人员（错误地）假设输入不会过长，因此（信任地）将输入复制到缓冲区中而产生的；由于输入实际上太长，它会溢出缓冲区，从而覆盖目标的内存。像下面这样无辜的代码可能是问题的根源：
![[Pasted image 20241127163609.png]]
在许多情况下，这种溢出并不会带来灾难性后果，例如，无意中向用户程序甚至操作系统提供错误输入可能会导致其崩溃，但不会更糟。但是，恶意程序员可以精心设计溢出缓冲区的输入，以便将自己的代码注入目标系统，本质上允许他们接管系统并执行自己的命令。如果成功攻击网络连接的用户程序，攻击者可以在受感染的系统上运行任意计算，甚至出租周期；如果成功攻击操作系统本身，攻击可以访问更多资源，这是一种所谓的特权升级（即用户代码获得内核访问权限）。如果你猜不到，这些都是坏事。

针对缓冲区溢出的第一个也是最简单的防御措施是阻止执行在地址空间的某些区域（例如，在堆栈内）内找到的任何代码。AMD 在其 x86 版本中引入的 NX 位（代表 No-eXecute）（英特尔现在提供类似的 XD 位）就是这样一种防御措施；它只是阻止在相应的页表条目中设置了此位的任何页面的执行。该方法可防止攻击者注入目标堆栈的代码被执行，从而缓解问题。

为了防御 ROP（包括其早期形式，返回 libc 攻击 [S+04]），Linux（和其他系统）添加了另一种防御措施，称为地址空间布局随机化 (ASLR)。操作系统不会将代码、堆栈和堆放置在虚拟地址空间内的固定位置，而是将它们的位置随机化，因此，编写实现此类攻击所需的复杂代码序列非常具有挑战性。因此，对易受攻击的用户程序的大多数攻击都会导致崩溃，但无法控制正在运行的程序。

## Other Security Problems: Meltdown And Spectre


# Summary
现在，您已经看到了对两个虚拟内存系统的从上到下的回顾。希望大部分细节都很容易理解，因为您应该已经很好地理解了基本机制和策略。有关 VAX/VMS 的更多详细信息，请参阅 Levy 和 Lipman 的优秀（简短）论文 [LL82]。我们鼓励您阅读它，因为它是了解这些章节背后源材料的绝佳方式。您还了解了一些有关 Linux 的知识。虽然它是一个庞大而复杂的系统，但它继承了过去的许多好主意，其中许多我们没有空间详细讨论。例如，Linux 在 fork() 时执行页面的惰性写时复制，从而通过避免不必要的复制来降低开销。Linux 还要求零页面（使用 /dev/zero 设备的内存映射），并且有一个后台交换守护进程 (swapd)，它将页面交换到磁盘以减少内存压力。事实上，VM 充满了从过去汲取的好点子，

