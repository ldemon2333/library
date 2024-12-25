Thus an operating system must fulfill three requirements: multiplexing, isolation, and interaction.

a monolithic kernel

xv6 在多核 RISC-V 微处理器上运行，其许多低级功能（例如，其进程实现）特定于 RISC-V。RISC-V 是 64 位 CPU，xv6 用“LP64”C 编写，这意味着 C 编程语言中的 long (L) 和指针 (P) 是 64 位，但 int 是 32 位。本书假设读者已经对某些架构进行了一些机器级编程，并将在出现时介绍 RISC-V 特定的想法。RISC-V 的一个有用参考是“RISC-V 阅读器：开放架构图集” [13]。用户级 ISA [2] 和特权架构 [1] 是官方规范。

Xv6 是为 qemu 的“-machine virt”选项模拟的支持硬件编写的。这包括 RAM、包含启动代码的 ROM、与用户键盘/屏幕的串行连接以及用于存储的磁盘。

# 2.1 Abstracting physical resources
遇到操作系统时，人们可能会问的第一个问题是为什么要有它？也就是说，可以将图 1.2 中的系统调用实现为一个库，应用程序可以与之链接。在这个计划中，每个应用程序甚至可以拥有自己的库，以满足其需求。应用程序可以直接与硬件资源交互，并以最适合应用程序的方式使用这些资源（例如，实现高或可预测的性能）。一些用于嵌入式设备或实时系统的操作系统就是以这种方式组织的。

这种库方法的缺点是，如果有多个应用程序正在运行，则这些应用程序必须表现良好。例如，每个应用程序必须定期放弃 CPU，以便其他应用程序可以运行。如果所有应用程序都相互信任且没有错误，这种合作式分时方案可能没问题。应用程序不信任彼此且有错误的情况更为常见，因此人们通常希望获得比合作方案提供的更强的隔离性。

为了实现强隔离，禁止应用程序直接访问敏感的硬件资源，而是将资源抽象为服务，这是很有帮助的。例如，Unix 应用程序仅通过文件系统的打开、读取、写入和关闭系统调用与存储交互，而不是直接读取和写入磁盘。这为应用程序提供了路径名的便利，并允许操作系统（作为接口的实现者）管理磁盘。即使隔离不是问题，有意交互（或只是希望彼此不干扰）的程序也可能会发现文件系统比直接使用磁盘更方便。

类似地，Unix 透明地在进程之间切换硬件 CPU，根据需要保存和恢复寄存器状态，这样应用程序就不必知道时间共享。这种透明度允许操作系统共享 CPU，即使某些应用程序处于无限循环中。

再举一个例子，Unix 进程使用 exec 来构建其内存映像，而不是直接与物理内存交互。这允许操作系统决定将进程放在内存中的何处；如果内存紧张，操作系统甚至可能将进程的一些数据存储在磁盘上。Exec 还为用户提供了文件系统存储可执行程序映像的便利。

Unix 进程之间的许多形式的交互都是通过文件描述符进行的。文件描述符不仅抽象出许多细节（例如，管道或文件中的数据存储在哪里），而且还以简化交互的方式定义。例如，如果管道中的一个应用程序失败，内核会为管道中的下一个进程生成 end-of-file 信号。

图 1.2 中的系统调用接口经过精心设计，既方便了程序员，又提供了强隔离的可能性。Unix 接口不是抽象资源的唯一方法，但事实证明它是一种非常好的方法。

# 2.2 User mode, supervisor mode, and system calls
为了实现进程隔离，RISC-V CPU在硬件上提供3种执行命令的模式：_machine mode_, _supervisor mode_, _user mode_。
1. machine mode的权限最高，CPU以machine mode启动，machine mode的主要目的是为了配置电脑，之后立即切换到supervisor mode。
2. supervisor mode运行CPU执行 _privileged instructions_，比如中断管理、对存储页表地址的寄存器进行读写操作、执行system call。运行在supervisor mode也称为在 _kernel space_中运行。
3. 应用程序只能执行user mode指令，比如改变变量、执行util function。运行在user mode也称为在 _user space_ 中运行。要想让CPU从user mode切换到supervisor mode，RISC-V提供了一个特殊的`ecall`指令，要想从supervisor mode切换到user mode，调用`sret`指令

# 2.3 Kernel organization
_monolithic kernel_：整个操作系统在kernel中，所有system call都在supervisor mode下运行。xv6是一个monolithic kernel。

A downside of the monolithic organization is that the interfaces between different parts of the OS are often complex. In a monolithic kernel, a mistake is fatal, because an error in supervisor mode will often cause the kernel to fail.

_micro kernel_：将需要运行在supervisor mode下的操作系统代码压到最小，保证kernel内系统的安全性，将大部分的操作系统代码执行在user mode下。
![[Pasted image 20241211134615.png]]
Figure 2.1 illustrates this microkernel design. For example, if an application like the shell wants to read or write a file, it sends a message to the file server and waits for a response.

OS such as Minix, L4, and QNX are organized as a mircrokernel with servers, and have seen wide deployment in embedded settings.

Xv6 is implemented as a monolithic kernel, like most Unix OS.
![[Pasted image 20241211135148.png]]

# 2.4 Code: xv6 organization
The xv6 kernel source is in the `kernel/` sub-directory. The inter-module interfaces are defined in `defs.h`.

# 2.5 Process overview
隔离的单元叫做进程，一个进程不能够破坏或者监听另外一个进程的内存、CPU、文件描述符，也不能破坏kernel本身。

为了实现进程隔离，xv6提供了一种机制让程序认为自己拥有一个独立的机器。一个进程为一个程序提供了一个私有的内存系统，或 _address space_，其他的进程不能够读/写这个内存。xv6使用 _page table_(页表)来给每个进程分配自己的address space，页表再将这些address space，也就是进程自己认为的虚拟地址(_virtual address_)映射到RISC-V实际操作的物理地址(_physical address_)

![[Pasted image 20241211135452.png]]
An address space includes the process's `user memory` starting at virtual address zero. Instructions come first, followed by global variables, then the stack, and finally a "heap" area (for malloc) that the process can expand as needed.

有许多因素限制了进程地址空间的最大大小：RISC-V 上的指针为 64 位宽；硬件在页表中查找虚拟地址时仅使用低 39 位；而 xv6 仅使用这 39 位中的 38 位。因此，最大地址为 2^38 − 1 = 0x3fffffffff，即 MAXVA（kernel/riscv.h:363）。在地址空间的顶部，xv6 为 trampoline 保留了一个页面，并保留了一个映射进程 trapframe 的页面。

Xv6 uses these two pages to transition into the kernel and back; the trampoline page contains the code to transition in and out of the kernel and mapping the trapframe is necessary to save/restore the state of the user process.  trampoline 作用户态到内核态的上下文切换，trapframe 保存用户态的寄存器。

p 是 `struct proc`
进程最重要的内核状态：1. 页表 `p->pagetable` 2. 内核堆栈`p->kstack` 3. 运行状态`p->state`，显示进程是否已经被分配、准备运行/正在运行/等待IO或退出

每个进程中都有线程(_thread_)，是执行进程命令的最小单元，可以被暂停和继续. To switch transparently between process, the kernel suspends the currently running thread and resumes another process's thread. 

每个进程有两个堆栈：用户堆栈(_user stack_)和内核堆栈(_kernel stack_ `p->kstack`)。当进程在user space中进行时只使用用户堆栈，当进程进入了内核(比如进行了system call)使用内核堆栈. When the process is executing user instructions, only its user stack is in use, and its kernel stack is empty. When the process enters the kernel (for a system call or interrupt), the kernel code executes on the process's kernel stack; while a  process is in the kernel, its user stack stll contains saved data, but isn's actively used. A process’s thread alternates between actively using its user stack and its kernel stack. The kernel stack is separate (and protected from user code) so that the kernel can execute even if a process has wrecked its user stack.

A process can make a system call by executing the RISC-V `ecall` instruction. This instruction raises the hardware privilege level and changes the program counter to a kernel-defined entry point. The code at the entry point switches to a kernel stack and executes the kernel instructions that implement the system call. When the system call completes, the kernel switches back to the user stack and returns to user space by calling the `sret` instruction, which lowers the hardware privilege level and resumes executing user instructions just after the system call instruction.

`p->state` indicates whether the process is allocated, ready to run, running, waiting for I/O, or exiting.

`p->pagetable` 保存进程的页表，格式符合 RISC-V 硬件的要求。xv6 使分页硬件在用户空间执行进程时使用进程的 `p->pagetable`。进程的页表还记录了分配给进程内存的物理页的地址。

总之，进程包含两个设计理念：an address space to give a process the illusion of its own memory, and, a thread, to give the process the illusion of its own CPU. In xv6, a process consists of one address space and one thread. 

# 2.6 Code: starting xv6, the first process and system call
RISC-V启动时，先运行一个存储于ROM中的bootloader程序`kernel.ld`来加载xv6 kernel到内存中，然后在machine模式下从`_entry`开始运行xv6。The RISC-V starts with paging hardware disableed: virtual addresses map directly to physical address. 

bootloader将xv6 kernel加载到0x80000000的物理地址中，因为前面的地址中有I/O设备

在`_entry`中设置了一个初始stack，`stack0`来让xv6执行`kernel/start.c`。在`start`函数先在machine模式下做一些配置，然后通过RISC-V提供的`mret`指令切换到supervisor mode，使program counter切换到`kernel/main.c`

Before jumping into supervisor mode, `start` performs one more task: it programs the clock chip to generate timer interrupts. With this housekeeping out of the way, `start` "returns" to supervisor mode by calling `mret`.

After `main` initializes several devices and subsystems, it creates the first process by calling `userinit`. The first process executes a small program written in RISC-V assembly, make the first system call in xv6 `initcode.S` loads the number for the `exec` system call, `SYS_exec`, into register `a7`, and then calls `ecall` to re-enter the kernel.

The kernel uses the number in register `a7` in `syscall` to call the desired system call.
The system call table maps `SYS_exec` to `sys_exec`, which the kernel invokes.

Once the kernel has completed `exec`, it returns to user space in the `/init` process.
`Init` creates a new console device file if needed and then opens it as file descriptors 0, 1, and 2. Then it starts a shell on the console.

`main`先对一些设备和子系统进行初始化，然后调用`kernel/proc.c`中定义的`userinit`来创建第一个用户进程。这个进程执行了一个`initcode.S`的汇编程序，这个汇编程序调用了`exec`这个system call来执行`/init`，重新进入kernel。`exec`将当前进程的内存和寄存器替换为一个新的程序(`/init`)，当kernel执行完毕`exec`指定的程序后，回到`/init`进程。`/init`(`user/init.c`)创建了一个新的console device以文件描述符0,1,2打开，然后在console device中开启了一个shell进程，至此整个系统启动了。

# 2.7 Security Model
您可能想知道操作系统如何处理错误或恶意代码。由于处理恶意代码比处理意外错误更困难，因此将此主题视为与安全相关是合理的。以下是操作系统设计中典型的安全假设和目标的高级视图。

操作系统必须假设进程的用户级代码将尽最大努力破坏内核或其他进程。用户代码可能会尝试取消引用其允许的地址空间之外的指针；它可能会尝试执行任何 RISC-V 指令，即使这些指令不是为用户代码设计的；它可能会尝试读取和写入任何 RISC-V 控制寄存器；它可能会尝试直接访问设备硬件；它可能会将巧妙的值传递给系统调用，以诱使内核崩溃或做一些愚蠢的事情。内核的目标是限制每个用户进程，以便它所能做的就是读取/写入/执行自己的用户内存，使用 32 个通用 RISC-V 寄存器，并以系统调用允许的方式影响内核和其他进程。内核必须阻止任何其他操作。这通常是内核设计中的绝对要求。

对内核自身代码的期望完全不同。内核代码被认为是由善意和细心的程序员编写的。内核代码应该没有错误，并且肯定不包含任何恶意代码。这个假设会影响我们分析内核代码的方式。例如，如果内核代码使用不当，许多内部内核函数（例如自旋锁）会导致严重问题。在检查任何特定的内核代码时，我们想要确保它的行为正确。但是，我们假设内核代码通常是正确编写的，并且遵循有关使用内核自身函数和数据结构的所有规则。在硬件级别，RISC-V CPU、RAM、磁盘等被认为按照文档中宣传的方式运行，没有硬件错误。

当然，在现实生活中，事情并非如此简单。很难防止聪明的用户代码通过消耗内核保护的资源（磁盘空间、CPU 时间、进程表槽等）使系统无法使用（或导致系统崩溃）。通常不可能编写无错误的代码或设计无错误的硬件；如果恶意用户代码的编写者知道内核或硬件错误，他们就会利用这些错误。有必要在内核中设计保护措施，以防出现错误：断言、类型检查、堆栈保护页等。最后，用户代码和内核代码之间的区别有时很模糊：一些特权用户级进程可能提供基本服务，并有效地成为操作系统的一部分，而在某些操作系统中，特权用户代码可以将新代码插入内核（如 Linux 的可加载内核模块）。