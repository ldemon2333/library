# 36.1 System Architecture
![[Pasted image 20241129132334.png]]
Some devices are connected to the system via a **general I/O bus**, which in many modern systems would be **PCI**; graphics and some other higher-performance I/O devices might be found here. Finally, even lower down are one or more of what we call a **peripheral bus**, such as SCSI, SATA, or USB. These connect slow devices to the system, including disks, mice, and keyboards.

Why do we need a hierarchical structure like this？Physics, and cost.

Modern systems increasingly use speciallized chipsets and faster point-to-point interconnects to improve performance. Figure 36.2 shows an approximate diagram of Intel's Z270 Chipset. 

The CPU connects to an I/O chip via Intel's proprietary **DMI (Direct Media Interface)**, and the rest of the device connect to this chip via a number of different interconnects. On the right, one or more hard drivers connect to the system via the **eSATA** interface;  **ATA** ( the AT Attachment, in reference to providing connection to the IBM PC AT), then **SATA** (for Serial ATA), and now **eSATA** (for external SATA) represent an evolution of storage interfaces over the past decades, with each step forward increasing performance to keep pace with modern storage devices.

Below the I/O chip are a number of **USB** (**Universal Serial Bus**) connections, which in this depiction enable a keyboard and mouse to be attached to the computer. On many modern systems, USB is used for low performance devices such as these.

Finally, on the left, other higher performance devices can be connected to the system via **PCIe (Peripheral Component Interconnect Express)**. In this diagram, a network interface is attached to the system here; higher performance storage devices (such as **NVMe** persistent storage devices) are often connected here.
![[Pasted image 20241129134148.png]]

# 36.2 A Canonical Device
![[Pasted image 20241129135359.png]]
We can see that a device has two important components. The first is the hardware **interface** it presents to the rest of the system.

The second part of any device is its **internal structure**. 设备的这一部分是特定于实现的，负责实现设备向系统呈现的抽象。非常简单的设备将有一个或几个硬件芯片来实现其功能；更复杂的设备将包括一个简单的 CPU、一些通用内存和其他设备专用芯片来完成它们的工作。例如，现代 RAID 控制器可能由数十万行固件（即硬件设备内的软件）组成，以实现其功能。

# 36.3 The Canonical Protocol
The device interface is comprised of three registers: a **status** register, which can be read to see the current status of the device; a **command** register, to tell the device to perform a certain task; and a **data** register to pass data to the device, or get data from the device. 

Let us now describe a typical interaction that the OS might have with the device in order to get the device to do something on its behalf. The protocol is as follows:
```
While (STATUS == BUSY)
	;//wait until device is not busy
Write data to DATA register
Write command to COMMAND register
	(starts the device and executes the command)
While (STATUS == BUSY)
	;//wait until device is done with your request
```
The protocol has four steps. In the first, the OS waits until the device is ready to receive a command by repeatedly reading the status register; we call this **polling** the device (basically, just asking it what is going on). Second, the OS sends some data down to the data register; one can imagine that if this were a disk, for example, that multiple writes would need to take place to transfer a disk block (say 4KB) to the device. When the main CPU is involved with the data movement, we refer to it as **programmed I/O (PIO)**. Third, the OS writes a command to the command register; doing so implicitly lets the device know that both data is present and that it should begin working on the command. Finally, the OS waits for the device to finish by again polling it in a loop, waiting to see if it is finished (it may then get an error code to indicates success of failure).

# 36.4 Lowering CPU Overhead With Interrupts
Use **interrupt**. When the device is finally finished with the operation, it will raise a hardware interrupt, causing the CPU to jump into the OS at a predetermined **interrupt service routine** (ISR) or more simply an **interrupt handler**.

Interrupts thus allow for **overlap** of computation and I/O, which is key for improved utilization. This timeline shows the problem:

Note that using interrupts is not always the best solution. For example, imagine a device that performs its tasks very quickly: the first poll usually finds the device to be done with task. Using an interrupt in this case will actually slow down the system: switching to another process, handling the interrupt, and switching back to the issuing process is expensive. Thus, if a device is fast, it may be best to poll; if it is slow, interrupts, which allow overlap, are best. If the speed of the device is not known, or sometimes fast and sometimes slow, it may be best to use a **hybrid** that polls for a little while and then, if the device is not yet finished, uses interrupts. This **two-phased** approach may achieve the best of both worlds.

## Tip: Interrupts not always better than polling
==Although interrupts allow for overlap of computation and I/O, they only really make sense for slow devices. Otherwise, the cost of interrupt handling and context switching may outweigh the benefits interrupts provide. There are also cases where a flood of interrupts may overload a system and lead it to livelock;==

Another reason not to use interrupts arises in networks. When a huge stream of incoming packets each generate an interrupt, it is possible for the OS to **livelock**, that is, find itself only processing interrupts and never allowing a user-level process to run and actually service the requests.

是的，这是网络中由于中断导致的典型 _活锁_（livelock）问题，尤其在高吞吐量系统中尤为严重。当系统受到大量网络数据包的冲击时，每个数据包都可能产生一个中断，通知操作系统去处理数据。然而，如果中断请求过多，操作系统可能会花费所有时间来处理这些中断，导致用户级进程无法运行。最终，系统可能会陷入一种状态，始终处理中断，却从未能够执行应用程序代码，从而进入所谓的“活锁”状态。

Another interrupt-based optimization is **coalescing**. 在这样的设置中，需要发出中断的设备首先要等待一段时间，然后再将中断传送到 CPU。在等待期间，其他请求可能很快就会完成，因此可以将多个中断合并为一个中断传送，从而降低中断处理的开销。当然，等待时间过长会增加请求的延迟，这是系统中常见的权衡。

# 36.5 More Efficient Data Movement With DMA
不幸的是，我们的规范协议还有另一个方面需要我们注意。特别是，当使用 programmed I/O (PIO) 将大量数据传输到设备时，CPU 再次因一个相当琐碎的任务而负担过重，从而浪费了大量的时间和精力，而这些时间和精力本可以用来运行其他进程。

The solution to this problem is something we refer to as **Direct Memory Access** (DMA). DMA 引擎本质上是系统内一种非常特殊的设备，它可以协调设备和主内存之间的传输，而无需太多 CPU 干预。

DMA works as follows. To transfer data to the device, for example, the OS would program the DMA engine by telling it where the data lives in memory, how much data to copy, and which device to send it to. At that point, the OS is done with the transfer and can proceed with other work. When the DMA is complete, the DMA controller raises an interrupt, and the OS thus know the transfer is complete.
![[Pasted image 20241129144136.png]]
From the timeline, you can see that the copying of data is now handled by the DMA controller. Because the CPU is free during that time, the OS can do something else, here choosing to run Process 2. Process 2 thus gets to use more CPU before Process 1 runs again.

# 36.6 Methods of Device Interaction
How to communicate with devices, How should the hardware communicate with a device? Should there be explicit instructions?

Two methods of device communication have developed. The first, oldest method (used by IBM mainframes for many years) is to have explicit **I/O instructions**. These instructions specify a way for the OS to send data to specific device registers and thus allow the construction of the protocols described above.

For example, on x86, the `in` and `out` instructions can be used to communicate with devices. To send data to a device, the caller specifies a register with the data in it, and a specific `port` which names the device. 

Such instructions are usually **privileged**. The OS controls devices, and the OS thus is the only entity allowed to directly communicate with them.

The second method to interact with devices is known as **memory-mapped I/O**. With this approach, the hardware makes device registers available as if they were memory locations. To access a particular register, the OS issues a load (to read) or store (to write) the address;

# 36.7 Fitting Into The OS: The Device driver
How to build a device-neutral OS?

Use abstraction. At the lowest level, a piece of software in the OS must know in datail how a device works. We call this piece of software a **device driver**, and any specifics of device interaction are encapsulated within.

![[Pasted image 20241129145653.png]]

Let us see how this abstraction might help OS design and implementation by examining the Linux file system software stack. Figure 36.4 is a rough and approximate depiction of the Linux software organization. As you can see from the diagram, a file system(Application) is completely oblivious to the specifics of which disk class it is using;
it simply issues block read and write requests to the generic block layer, which routes them to the appropriate device driver, which handles the details of issuing the specific request.

The diagram also shows a **raw interface** to devices, which enables special applications (such as a **file-sytem checker** or a **disk defragmentation tool**) to directly read and write blocks without using the file abstraction. Most systems provide this type of interface to support these low-level storage management applications.

Note that the encapsulation seen above can have its downside as well. For example, if there is a device that has many special capabilities, but has to present a generic interface to the rest of the kernel, those special capabilities will go unused. This situation arises, for example, in Linux with SCSI devices, which have very rich error reporting; because other block devices (e.g., ATA/IDE) have much simpler error handling, all that higher levels of software ever receive is a generic EIO (generic IO error) error code; any extra detail that SCSI may have provided is thus lost to the file system.

# 36.8 Case Study: A Simple IDE Disk Driver
为了更深入地了解，让我们快速看一下实际的设备：IDE 磁盘驱动器。我们总结了此参考文献中描述的协议；我们还将查看 xv6 源代码，以了解一个工作 IDE 驱动程序的简单示例。
![[Pasted image 20241129150744.png]]
An IDE disk presents a simple interface to the system, consisting of  four types of register: control, command block, status, and error. These registers are available by reading or writing to specific "I/O addresses" (such as **0x3F6** below) using (on x86) the `in` and `out` I/O instructions.

The basic protocol to interact with the device is as follows, assuming it has already been initialized.
- **Wait for drive to be ready**. Read Status Register (0x1F7) until drive is READY and not BUSY.
- **Write parameters to command registers**. Write the sector count, logical block address (LBA) of the sectors to be accessed, and drive number (master=0x00 or slave=0x10, as IDE permits just two drivers)  to command registers (0x1F2-0x1F6).
- **Start the I/O**. Write READ | WRITE command to command register (0x1F7).
- **Data transfer (for writes)**: Wait until drive status is READY and DRQ (drive request for data); write data to data port.
- **Handle interrupts**. 
- **Error handling**.

![[Pasted image 20241129152000.png]]

该协议的大部分内容可以在 xv6 IDE 驱动程序（图 36.6）中找到，该驱动程序（初始化后）通过四个主要函数工作。第一个是 ide_rw()，它将请求排队（如果有其他待处理请求），或者直接将其发送到磁盘（通过 ide_start_request()）；在任一情况下，例程都会等待请求完成，并且调用进程将进入睡眠状态。第二个是 ide_start_request()，它用于将请求（在写入的情况下可能是数据）发送到磁盘；in 和 out x86 指令分别用于读取和写入设备寄存器。the start request 例程使用第三个函数 ide_wait_ready() 来确保驱动器在向其发出请求之前已准备就绪。最后，当发生中断时，将调用 ide_intr()；它从设备读取数据（如果请求是读取而不是写入），唤醒等待 I/O 完成的进程，并且（如果 I/O 队列中有更多请求），通过 ide_start_request() 启动下一个 I/O。

# 36.10 Summary
现在您应该对操作系统如何与设备交互有了非常基本的了解。本文引入了两种技术，即中断和 DMA，以提高设备效率，并描述了两种访问设备寄存器的方法，即显式 I/O 指令和内存映射 I/O。最后，介绍了设备驱动程序的概念，展示了操作系统本身如何封装低级细节，从而使得以设备中立的方式构建操作系统的其余部分变得更加容易。






