Write an xv6 device driver for a network interface card (NIC).

# Background
On this emulated LAN, xv6 (the "guest") has an IP address of 10.0.2.15. Qemu also arranges for the computer running qemu to appear on the LAN with IP address 10.0.2.2. When xv6 uses the E1000 to send a packet to 10.0.2.2, qemu delivers the packet to the appropriate application on the (real) computer on which you're running qemu (the "host").

The Makefile configures QEMU to record all incoming and outgoing packets to the file `packets.pcap` in your lab directory. To display the recorded packets:
```
tcpdump -XXnr packets.pcap
```

The file `kernel/e1000.c` contains initialization code for the E1000 as well as empty functions for transmitting and receiving packets, which you'll fill in.
`kernel/e1000_dev.h` contains definitions for registers and flag bits defined by the E1000 and described in the Inter E1000 [Software Developer's Manual](https://pdos.csail.mit.edu/6.828/2021/readings/8254x_GBe_SDM.pdf). kernel/net.c and `kernel/net.h` contain a simple network stack that implements the [IP](https://en.wikipedia.org/wiki/Internet_Protocol), [UDP](https://en.wikipedia.org/wiki/User_Datagram_Protocol), and [ARP](https://en.wikipedia.org/wiki/Address_Resolution_Protocol) protocols. These files also contain code for a flexible data structure to hold packets, called an `mbuf`. Finally, `kernel/pci.c` contains code that searches for an E1000 card on the PCI bus when xv6 boots.

# Your Job
Your job is to complete `e1000_transmit()` and `e1000_recv()`, both in `kernel/e1000.c`, so that the driver can transmit and receive packets. 

熟悉系统驱动与外围设备的交互、内存映射寄存器与 DMA 数据传输，实现与 E1000 网卡交互的核心方法：transmit 与 recv。

本 lab 的难度主要在于阅读文档以及理解 CPU 与操作系统是如何与外围设备交互的。换言之，更重要的是理解概念以及 lab 已经写好的模版代码的作用。