# Page fault 概述
Lazy allocation of user-space heap memory. Xv6 applications ask the kernel for heap memory using the `sbrk()` system call. In the kernel we've given you, `sbrk()` allocates physical memory and maps it into the process's virtual address space. It can take a long time for a kernel to allocate and map memory for a large request. To allow `sbrk()` to complete more quickly in these cases, sophisticated kernels allocate user memory lazily. That is, `sbrk()` doesn't allocate physical memory, but just remembers which user addresses are allocated and marks those addresses as invalid in the user page table. ==When the process first tries to use any given page of lazily-allocated memory, the CPU generates a page fault,== which the kernel handles by allocating physical memory, zeroing it, and mapping it. 

**RISC-V 架构**：在 RISC-V 中，也没有专门的指令来显式触发 page fault，但与内存相关的访问操作（如 `LW`, `SW`）会在出现缺页或权限错误时触发 **Page Fault** 异常，控制权转交给操作系统的异常处理程序。对于页错误，RISC-V 使用 **trap** 来中断当前程序执行。

当程序访问一个虚拟内存中的 page 时，发现该地址无效（可能是在页表中找不到，或者是找到之后 PTE_V 标志位为 0）。

page fault 的提供以下信息：
- 出错的虚拟地址，当出现 page fault 的时候，是由于页表中找不到这个地址对应的 PTE 或者 PTE 无效，这里的出错的虚拟地址值得就是这条"找不到的地址"，XV6 内核会打印出错的虚拟地址，并且这个地址会被保存在 **STVAL 寄存器**中。
- 出错的原因，查看 RISC-V 文档可知有三种引起 page fault 的原因：**load 引起的 page fault**；**store 引起的 page fault**；**指令执行引起的 page fault**。这个信息存在 **SCAUSE 寄存器**中，总共有3个类型的原因与page fault 相关，分别是读、写和指令
- 触发 page fault 的指令的地址，即 page fault 在**用户空间**发生的位置。作为trap处理代码的一部分，这个地址存放在 **SEPC 寄存器**中，并同时会保存在trapframe->epc 中。

借助 Page fault 可以实现以下功能：
- lazy page allocation
- copy-on-write fork（会作为 lab 出现）
- demand paging
- memory mapped files（会作为 lab 出现）

为什么借助 pf 可以实现这些功能，**归根到底还是 pf 会导致 trap 到 kernel mode**，在 kernel mode 中，就可以做很多“魔法”。下面会依次介绍这些 pf 的处理方式。

## Lazy allocation
XV6提供了一个系统调用叫 sbrk，这个系统调用的核心实现如下：
![[Pasted image 20241220223802.png]]

在XV6中，sbrk的实现默认是 **eager allocation**。即一旦调用了sbrk，内核会**立即分配**应用程序所需要的物理内存。但是实际上，对于应用程序来说很难预测自己需要多少内存，所以通常来说，应用程序倾向于申请多于自己所需要的内存。以矩阵计算程序为例，程序员需要为最坏的情况做准备，比如说为最大可能的矩阵分配内存，但是应用程序可能永远也用不上这些内存。所以，**程序员过多的申请内存但是过少的使用内存，这种情况还挺常见的**。

所以就引出了lazy allocation 的概念。核心思想非常简单，摒弃 eager allocation，sbrk 被调用时不会立即分配内存，只是记录一下"假如真的分配了内存，那么现在应用程序可用的内存是多少"（在实际的 xv6 中，这个值是由 p->sz 记录的，他表示**堆顶指针**）
![[Pasted image 20241220223857.png]]
当应用程序真的用到了**新申请的这部分内存**，由于没有分配、页表中没有映射，自然找不到相应PTE，这时会触发page fault，但是 kernel 会识别到：要访问 va **小于新的 p->sz**，并且**大于旧的 p->sz**，就知道这是一个当初假装分配的地址，所以这时才会真正分配物理地址并且在用户程序的页表中添加 PTE，所以在 page fault handler 中就会：
- 在page fault handler中，通过kalloc函数分配一个内存page；并初始化这个 page 内容为0；
- 将这个内存 page 映射到 user page table 中；
- 最后重新执行指令（SEPC 寄存器记录了发生 pf 时的地址，所以可以回到“事发地”重新执行指令）
总之，lazy allocation 的核心概念就是“**将分配物理内存 page 推迟到了真正访问这个内存 page 时做**”。

## Zero Fill On Demand
再次搬出这张图：一个用户程序的内存分布图
![[Pasted image 20241220224106.png]]
但是这张图省略了一点布局，就是除了 text 区域，data 区域，同时还有一个BSS区域，BSS 区域位于 data 区域之后。text 区域存放是**程序的指令**，data 区域存放的是**初始化了的全局变量**，BSS 包含了**未被初始化**或者**初始化为0**的全局变量。

之所以要创建这个 BSS 区域，是因为作为全局变量，元素初始值都是 0 的情况还是蛮多的，比如在 C 语言中定义了一个**大**的矩阵作为全局变量，那么它的元素初始值都是 0。由于 BSS 里面保存了未被初始化的全局变量，这里可能有很多 page，但是所有的page内容都为0。每个 page 都需要分配实际的物理内存空间：
![[Pasted image 20241220224138.png]]
但如果采用 Zero Fill On Demand ，那么 BSS 区域的 page 就不需要全部到不同的物理内存上，而是都映射到同一个物理 page 上，之所以能这么做，是因为所有的 page 内容都是 0，所以就可以用一个 page 代替其他 page，**但是由于共享了 page，便不能随意写这个 page 了，所以这个 page 的 flag 标志位设置为只读**：

![[Pasted image 20241220224208.png]]
如果需要写这个BSS 区域的某个 page（va），由于设置了只读，所以会触发 page fault，这时在 page fault handler 中就会：
- 分配一个新的物理 page，将 va 映射到新的 page 上
- 在新的 page 上执行写指令，而 BSS 区域其他page 依旧映射到原 page 上
![[Pasted image 20241220224248.png]]
## Copy on write fork
当父进程或者子进程写这些共享的地址时，就会触发 page fault，page fault handler 就会

1. 复制出错的 page 并重新映射
    
2. 在新 page 上写
    
3. 将复制的 page 和原 page 的标志位修改为可写（原来是只读）

![[Pasted image 20241220224715.png]]
关于 COW fork 还有两个重要的细节：
1. 当发生page fault时，我们其实是在向一个只读的地址执行写操作。内核如何能分辨现在是一个 copy-on-write fork 的场景，而不是应用程序在向一个正常的只读地址写数据？

这其实是个共性的问题

**在 lazy allocation 中**，我们如何知道是向 PTE 找不到是因为本该分配的 lazy 了，还是确实没找到？答案是根据访问地址判断：如果要访问的 va < 新的 p->sz 且 > 旧的 p->sz，说明是 lazy 的情况。

**在 zero fill on demand 中**，如何能分辨现在是一个 zero fill on demand 的场景，而不是应用程序在向一个正常的只读地址写数据？答案也是根据 va 判断，va 是否是 BSS 段的 page，如果是则说明是一个 zero fill on demand 的场景

**那么在 COW fork 中也一样**，还记得 RISC-V 中一条 PTE 有10 bit 的辅助位把，其中 8、9bit 是保留位，我们可以使用其中的任意一个 bit 作为“**这是 COW page** ”的标记：

![[Pasted image 20241220224829.png]]
2. 第二个细节是 page 释放时要小心翼翼，因为共享物理 page 的存在，每次释放 page 时都要确保没有进程引用这些 page，所以需要有一个引用计数器来统计当前有多少个进程在使用这个 page，只有引用计数器为 0，才可以释放 page
## Demand paging

## Memory Mapped Files

# Eliminate allocation from sbrk()
You first task is to delete page allocation from the `sbrk` system call implementation. You new `sbrk(n)` should just increment the process's size (`myproc()->sz`) by n and return the old size. It should not allocate memory.

# Lazy allocation
Modify the code in `trap.c` to respond to a page fault from user space by mapping a newly-allocated page of physical memory at the faulting address, and then returning back to user space to let the process continue executing. 

Hints:
- Seeing if `r_scause()` is 13 or 15 in `usertrap()`.
- `r_stval()` returns the RISC-V `stval` register, which contains the virtual address that caused the page fault.
- `uvmunmap()` will panic; modify it to not  panic if some pages aren't mapped.
- If the kernel crashes, look up `sepc` in `kernel/kernel.asm`.
- Use your `vmprint` function from pgtbl lab to print the content of a page table.
- If you see the error "incomplete type proc", include `spinlock.h` then `proc.h`.
If all goes well, you lazy allocation code should result in `echo hi` working. You should get at least one page fault (and thus lazy allocation), and perhaps two.

# Lazytests and Usertests
- Handle negative `sbrk()` arguments.
- Kill a process if it page-faults on a virtual memory address higher than any allocated with `sbrk()`.
- Handle the parent-to-child memory copy in `fork()` correctly.
- Handle the case in which a process passes a valid address from `sbrk()` to a system call such as read or write, but the memory for that address has not yet been allocated.
- Handle out-of-memory correctly: if `kalloc()` fails in the page fault handler, kill the current process.
- Handle faults on the invalid page below the user stack.