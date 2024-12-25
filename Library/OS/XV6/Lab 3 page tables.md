# Speed up system calls
某些操作系统（例如 Linux）通过在用户空间和内核之间共享只读区域中的数据来加速某些系统调用。这消除了执行这些系统调用时内核交叉的需要。为了帮助您了解如何将映射插入页表，您的首要任务是为 xv6 中的 getpid() 系统调用实现此优化。

创建每个进程时，在 USYSCALL（memlayout.h 中定义的 VA）处映射一个只读页面。在此页面的开头，存储一个 struct usyscall（也在 memlayout.h 中定义），并将其初始化为存储当前进程的 PID。对于此实验，ugetpid() 已在用户空间端提供，并将自动使用 USYSCALL 映射。如果在运行 pgtbltest 时 ugetpid 测试用例通过，您将获得此部分实验的全部学分。

hints:
- You can perform the mapping in proc_pagetable() in kernel/proc.c.
- Choose permission bits that allow userspace to only read the page.
- You may find that mappages() is a useful utility.
- Don't forget to allocate and initialize the page in allocproc().
- Make sure to free the page in freeproc().

# print a page table
To help you visualize RISC-V page tables, and perhaps to aid future debugging, your second task is to write a function that prints the contents of a page table.

定义一个名为 vmprint() 的函数。它应该接受一个 pagetable_t 参数，并以下面描述的格式打印该页表。在 exec.c 中的 return argc 之前插入 if(p->pid\==1) vmprint(p->pagetable)，以打印第一个进程的页表。如果您通过了 make grade 的 pte 打印输出测试，您将获得此部分实验的全部学分。

Now when you start xv6 it should print output like this, describing the page table of the first process at the point when it has just finished exec()ing init:

The first line displays the argument to vmprint. After that there is a line for each PTE, including PTEs that refer to page-table pages deeper in the tree. Each PTE line is indented by a number of " .." that indicates its depth in the tree. Each PTE line shows the PTE index in its page-table page, the pte bits, and the physical address extracted from the PTE. Don't print PTEs that are not valid. In the above example, the top-level page-table page has mappings for entries 0 and 255. The next level down for entry 0 has only index 0 mapped, and the bottom-level for that index 0 has entries 0, 1, and 2 mapped.

第一行显示 vmprint 的参数。之后，每个 PTE 都有一行，包括引用树中较深的页表页的 PTE。每个 PTE 行都缩进多个“ ..”，表示其在树中的深度。每个 PTE 行都显示其页表页中的 PTE 索引、pte 位以及从 PTE 中提取的物理地址。不要打印无效的 PTE。在上面的示例中，顶级页表页具有条目 0 和 255 的映射。条目 0 的下一级仅映射了索引 0，而该索引 0 的底层映射了条目 0、1 和 2。

Your code might emit different physical addresses than those shown above. The number of entries and the virtual addresses should be the same.

Some hints:

- You can put vmprint() in kernel/vm.c.
- Use the macros at the end of the file kernel/riscv.h.
- The function freewalk may be inspirational.
- Define the prototype for vmprint in kernel/defs.h so that you can call it from exec.c.
- Use %p in your printf calls to print out full 64-bit hex PTEs and addresses as shown in the example.
实现一个 `vmprint()` 函数，该函数接收一个 pagetable_t 的参数，然后打印该页表，具体格式参考图片中的样式。在创建 `init` 进程时，调用这个函数打印页表。


# Detecting which pages have been accessed
一些垃圾收集器（一种自动内存管理形式）可以从有关哪些页面已被访问（读取或写入）的信息中受益。在本实验的这一部分中，您将向 xv6 添加一项新功能，该功能通过检查 RISC-V 页表中的访问位来检测并向用户空间报告此信息。每当 RISC-V 硬件页面遍历器解决 TLB 未命中时，它都会在 PTE 中标记这些位。

您的任务是实现 pgaccess()，这是一个报告已访问过哪些页面的系统调用。该系统调用需要三个参数。首先，它需要检查的第一个用户页面的起始虚拟地址。其次，它需要检查的页面数。最后，它需要缓冲区的用户地址，以将结果存储到位掩码（一种每页使用一位的数据结构，其中第一页对应于最低有效位）。如果在运行 pgtbltest 时 pgaccess 测试用例通过，您将获得此部分实验的全部学分。

Some hints:

- Start by implementing sys_pgaccess() in kernel/sysproc.c.
- You'll need to parse arguments using argaddr() and argint().
- For the output bitmask, it's easier to store a temporary buffer in the kernel and copy it to the user (via copyout()) after filling it with the right bits.
- It's okay to set an upper limit on the number of pages that can be scanned.
- walk() in kernel/vm.c is very useful for finding the right PTEs.
- You'll need to define PTE_A, the access bit, in kernel/riscv.h. Consult the RISC-V manual to determine its value.
- Be sure to clear PTE_A after checking if it is set. Otherwise, it won't be possible to determine if the page was accessed since the last time pgaccess() was called (i.e., the bit will be set forever).
- vmprint() may come in handy to debug page tables.
实现一个 `pgaccess()` 函数，这个函数的申明为：`int pgaccess(void *base, int len, void *mask);`。这个函数的主要作用就是检测**从上次调用这个函数开始**，页表是否被访问过。其中 `base` 参数是要检测的第一个页表，`len` 从这个页表开始，要检测多少个页表，而我们需要把每个页表的访问情况写到 `mask` 上。这个 `mask` 的作用和 lab2 中的 trace_mask 相同，如果当前页表被访问，那么 `mask` 中对应的位应该是 1。

# A kernel page table per process
Xv6 has a single kernel page table that's used whenever it executes in the kernel. The kernel page table is a direct mapping to physical addresses, so that kernel virtual address x maps to physical address x. Xv6 also has a separate page table for each process's user address space, containing only mappings for that process's user memory, starting at virtual address zero. 由于内核页表不包含这些映射，因此用户地址在内核中无效。因此，当内核需要使用在系统调用中传递的用户指针（例如，传递给 write() 的缓冲区指针）时，内核必须首先将该指针转换为物理地址。使用 copyin 函数，把用户页表的映射搬入到内核空间中。本节和下一节的目标是允许内核直接取消引用用户指针。


您的第一项工作是修改内核，以便每个进程在内核中执行时都使用自己的内核页表副本。修改 struct proc 以维护每个进程的内核页表，并修改调度程序以在切换进程时切换内核页表。对于此步骤，每个进程的内核页表应与现有的全局内核页表相同。如果用户测试运行正常，则您通过了实验的这一部分。

阅读本作业开头提到的书籍章节和代码；了解虚拟内存代码的工作原理后，修改起来会更容易。页表设置中的错误可能会因缺少映射而导致陷阱，可能会导致加载和存储影响物理内存的意外页面，还可能导致从错误的内存页面执行指令。

Some hints:
