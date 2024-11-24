The simple approach of using a base and bounds register pair to virtualize memory is wasteful. It also makes it quite hard to run a program when the entire address space doesn't fit into memory.

# 16.1 Segmentation: Generalized Base/Bounds
The idea is simple: instead of having just one base and bounds pair in our MMU, why not have a base and bounds pair per logical **segment** of the address space? A segment is just a contiguous portion of the address space of a particular length, and in our canonical address space, we have logically-different segments: code, stack, and heap. What segmentation allows the OS to do is to place each one of those segments in different parts of physical memory, and thus avoid filling physical memory with unused virtual address space.

![[Pasted image 20241118205045.png]]

As you can see in the diagram, only used memory is allocated space in physical memory.

The hardware structure in our MMU required to support segmentation is just what you'd expect: in this case, a set of three base and bounds register pairs. 

![[Pasted image 20241118205325.png]]

非法访问，越出该数据段的Size，触发 segmentation fault。

# 16.2 Which Segment Are We Referring To?
The hardware uses segment registers during translation. How does it know the offset into a segment, and to which segment an address refers?

![[Pasted image 20241118205843.png]]

You may also have noticed that when we use the top two bits, and we only have three segments (code, heap, stack), one segment of the address space goes unused. To fully utilize the virtual address space (and avoid an unused segment), some systems put code in the same segment as the heap and thus use only one bit to select which segment to use.

Another issue with using the top so many bits to select a segment is that it limits use of the virtual address space. Specifically, each segment is limited to a *maximum size*, which in our example is 4KB (using the top two bits to choose segments implies the 16KB address space gets chopped into four pieces, or 4KB in this example). If a running program wishes to grow a segment (say the heap, or the stack) beyond that maximum, the program is out of luck.

# 16.3 What about the stack?
The stack has been relocated to physical address 28KB in the diagram above, but with one critical difference: *it grows backwards*. In physical memory, it "starts" at 28KB and grows back to 26KB, corresponding to virtual addresses 16KB to 14KB; translation must proceed differently.

![[Pasted image 20241118211636.png]]

The first thing we need not only base and bounds values, the hardware also needs to know which way the segment grows (a bit, for example, that is set to 1 when the segment grows in the positive direction, and 0 for negative).

# 16.4 Support for Sharing
code sharing 机制, 硬件上增加 protection bits. While each process still thinks that it is accessing its own private memory, the OS is secrectly sharing memory which cannot be modified by the process, and thus the illusion is preserved. 映射到同一个物理页上。

![[Pasted image 20241118213246.png]]

With protection bits, the hardware algorithm described earlier would also have to change. In addition to checking whether a virtual address is within bounds, the hardware also has to check whether a particular access is permissible. If a user process tries to write to a read-only segment, or execute from a non-executable segment, the hardware should raise an exception, and thus let the OS deal with the offending process.

# 16.5 Fine-grained vs. Coarse-grained Segmentation
更加细粒度的分段需要使用 **segment table** of some kind stored in memory. 

# 16.6 OS Support
segmentation raises a number of new issues for the operating system. The first is an old one: what should the OS do on a context switch? You should have a good guess by now: the segment registers must be saved and restored. Clearly, each process has its own virtual address space, and the OS must make sure to set up these registers correctly before letting the process run again.

The second is OS interaction when segments grow (or perhaps shrink). For example, a program may call *malloc()* to allocate an object. The heap segment itself may need to grow. ( sbrk() system call ). Do note that the OS could reject the request, if no more physical memory is available, or if it decides that the calling process already has too much.

The last, issue is managing free space in physical memory. When a new address space is created, the OS has to be able to find space in physical memory for its segments. Previously, we assumed that each address space was the same size, and thus physical memory could be thought of as a bunch of slots where processes would fit in. Now, we have a number of segments per process, and each segment might be a different size.

The general problem that arises is that physical memory quickly becomes full of little holes of free space, making it difficult to allocate new segments, or to grow existing ones. We call this problem **external fragmentation**.

One solution to this problem would be to **compact** physical memory by rearranging the existing segments. For example, the OS could stop whichever processes are running, copy their data to one contiguous region of memory, change their segment register values to point to the new physical locations, and thus have a large free extent of memory with which to work. By doing so, the OS enables the new allocation request to succeed. 但是非常耗时。都会存在大量的外部碎片。

![[Pasted image 20241118215308.png]]

# 16.7 Summary
也许更重要的问题是，分段仍然不够灵活，无法支持我们完全通用的稀疏地址空间。例如，如果我们在一个逻辑段中有一个大但使用稀疏的堆，则整个堆必须仍驻留在内存中才能被访问。换句话说，如果我们的地址空间使用模型与底层分段设计支持它的方式不完全匹配，分段就不能很好地工作。
