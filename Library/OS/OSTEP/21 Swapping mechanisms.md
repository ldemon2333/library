How to go beyond physical memory, how can the OS make use of a larger, slower device to transparently provide the illusion of a large virtual address space?

Beyond just a single process, the addition of swap space allows the OS to support the illusion of a large virtual memory for multiple concurrently running processes.

# 21.1 Swap Space
交换区在磁盘上
The first thing we will need to do is to reserve some space on the **disk** for moving pages back and forth. In operating systems, we generally refer to such space as `swap space`, because we `swap` pages out of memory to it and `swap` pages into memory from it. Thus, we will simply assume that the OS can read from and write to the swap space, in page-sized units. To do so, the OS will need to remember the `disk address` of a given page.

The size of the swap space is important, as utimately it determines the maximum number of memory pages that can be in use by a system at a given time.

在这个小例子（图 21.1）中，你可以看到一个 4 页物理内存和一个 8 页交换空间的小例子。在这个例子中，三个进程（Proc 0、Proc 1 和 Proc 2）正在积极共享物理内存；然而，这三个进程中只有部分有效页面在内存中，其余页面位于磁盘上的交换空间中。第四个进程（Proc 3）将其所有页面都交换到磁盘，因此显然当前没有运行。一个交换块保持空闲。即使从这个小例子中，希望你也能明白如何使用交换空间让系统假装内存比实际更大。

![[Pasted image 20241126190802.png]]
我们应该注意，交换空间并不是交换流量的唯一磁盘位置。例如，假设您正在运行程序二进制文件（例如，ls 或您自己编译的主程序）。此二进制文件的代码页最初位于磁盘上，当程序运行时，它们被加载到内存中（在程序开始执行时一次全部加载，或者像在现代系统中一样，在需要时一次加载一页）。但是，如果系统需要在物理内存中腾出空间以满足其他需求，它可以安全地重新使用这些代码页的内存空间，因为它知道它稍后可以从文件系统中的磁盘二进制文件中再次交换它们。

# 21.2 The Present Bit
Remember that the hardware first extracts the VPN from the virtual address, checks the TLB for a match (a TLB hit), and if a hit, produces the resulting physical address and fetches it from memory. This is hopefully the common case, as it is fast (不需要额外访存).

If the VPN is not found in the TLB, the hardware locates the page table in memory (using the **page table base register**) and looks up the **page table entry** for this page using the VPN as an index. If the page is valid and present in physical memory, the hadrware extracts the PFN from the PTE, installs it in the TLB, and retries the instruction, this time generating a TLB hit.

If we wish to allow pages to be swapped to disk, however, we must add even more machinery. Specifically, when the hardware looks in the PTE, it may find that the page is not present in physical memory. By checking present bit (访问位). If the present bit is set to one, it means the page is present in physical memory and everything proceeds as above; if it is set to zero, the page is not in memory but rather on disk somewhere. The act of accessing a page that is not in physical memory is commonly referred to as a **page fault**.

# 21.3 The Page Fault
If a page is not present and has been swapped to disk, the OS will need to swap the page into memory in order to service the page fault. Thus, how will the OS know where to find the desired page? In many systems, the page table is a natural place to store such information. Thus, the OS could use the bits in the PTE normally used for data such as the PFN of the page for a disk address. When the OS receives a page fault for a page, it looks in the PTE to find the address, and issues the request to disk to fetch the page into memory.

当磁盘 I/O 完成时，操作系统将更新页表以将页面标记为存在，更新页表条目 (PTE) 的 PFN 字段以记录新获取的页面在内存中的位置，然后重试该指令。下一次尝试可能会产生 TLB 未命中，然后会对其进行服务并使用转换更新 TLB（可以在服务页面错误时交替更新 TLB 以避免此步骤）。最后，最后一次重新启动将在 TLB 中找到转换，然后继续从转换后的物理地址处的内存中获取所需的数据或指令。

请注意，在 I/O 正在进行时，该进程将处于阻塞状态。因此，在服务页面错误时，操作系统可以自由运行其他就绪进程。由于 I/O 成本高昂，因此一个进程的 I/O（页面错误）与另一个进程的执行重叠是多程序系统最有效地利用其硬件的另一种方式。

# 21.4 What If Memory Is Full?
在上述过程中，您可能会注意到，我们假设有足够的可用内存来从交换空间中调入页面。当然，情况可能并非如此；内存可能已满（或接近满）。因此，操作系统可能希望首先调出一个或多个页面，为操作系统即将引入的新页面腾出空间。选择要踢出或替换的页面的过程称为页面替换策略。

# 21.5 Page Fault Control Flow
![[Pasted image 20241126194129.png]]
![[Pasted image 20241126194140.png]]

First, that the page was both `present` and `valid`（18-21）

从图 21.3 中的软件控制流中，我们可以看到操作系统大致必须做什么才能处理页面错误。首先，操作系统必须找到一个physical fram ，让即将发生页面驻留在其中；如果没有这样的页面，我们将不得不等待替换算法运行并将一些页面踢出内存，从而释放它们以供在此处使用。有了物理框架，处理程序就会发出 I/O 请求，从交换空间读取页面。最后，当这个缓慢的操作完成时，操作系统会更新页表并重试该指令。重试将导致 TLB 未命中，然后在另一次重试时，TLB 命中，此时硬件将能够访问所需的项目。

# 21.6 When Replacements Really Occur
为了保持少量内存可用，大多数操作系统都具有某种高水位 (HW) 和低水位 (LW)，以帮助决定何时开始从内存中清除页面。其工作原理如下：当操作系统注意到可用的页面少于 LW 时，负责释放内存的后台线程就会运行。该线程清除页面，直到有可用的 HW 页面。然后，后台线程（有时称为**swap daemon** or **page daemon**）进入休眠状态，很高兴它已经释放了一些内存供正在运行的进程和操作系统使用。

通过同时执行多个替换，可以实现新的性能优化。例如，许多系统将对多个页面进行聚类或分组，并将它们同时写入交换分区，从而提高磁盘的效率；正如我们稍后在更详细地讨论磁盘时将看到的那样，这种聚类减少了磁盘的寻道和旋转开销，从而显著提高了性能。

为了与后台分页线程一起工作，应该稍微修改图 21.3 中的控制流；算法不是直接执行替换，而是简单地检查是否有任何可用的空闲页面。如果没有，它会通知后台分页线程需要空闲页面；当线程释放一些页面时，它会重新唤醒原始线程，然后原始线程可以分页所需的页面并开始工作。

# 21.7 Summary
Introduced the notion of accessing more memory than is physically present within a system. To do so requires more complexity in page-table structures, as a `present bit` must be included to tell us whether the page is present in memory or not. When not, the operating system `page-fault handler` runs to service the page fault.