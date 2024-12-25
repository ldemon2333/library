Three kinds of event:
- system call, when a user program executes the `ecall` instruction to ask the kernel to do something for it.
- Exception, an instruction (user or kernel) does something illegal.
- device interrupt, when a device signals that it needs attention, for example when the disk hardware finishes a read or write request.

This book uses `trap` as a generic term for these situations. The usual sequence is that a trap forces a transfer of control into the kernel; the kernel saves registers and other state so that execution can be resumed.

Xv6 handles all traps in the kernel; traps are not delivered to user code. 对于系统调用来说，在内核中处理陷阱是很自然的。对于中断来说，这是有意义的，因为隔离要求只允许内核使用设备，并且内核是一种方便的机制，可以在多个进程之间共享设备。对于异常来说，这也是有意义的，因为 xv6 通过终止有问题的程序来响应来自用户空间的所有异常。

Xv6 trap handling proceeds in four stages: hardware actions taken by the RISC-V CPU, some assembly instructions that prepare the way for kernel C code, a C function that decides what to do with the trap, and the system call or device-driver service routine. 虽然三种陷阱类型之间的共性表明内核可以使用单个代码路径处理所有陷阱，但事实证明，为三种不同情况使用单独的代码会很方便：来自用户空间的陷阱、来自内核空间的陷阱和计时器中断。处理陷阱的内核代码（汇编程序或 C）通常称为处理程序；第一个处理程序指令通常用汇编程序（而不是 C）编写，有时称为向量。

ecall(environment call)指令负责==提升RISCV的优先级模式==，RISCV有三种模式：**用户模式(User Mode)、监视者模式(Supervisor Mode)和机器模式(Machine Mode)**。这三种模式的优先级依次升高，当我们主动调用一个系统调用时时，需要提升CPU的特权模式，以获得==对某些寄存器的访问权==和==某些指令的执行权==。

事实上，ecall主动触发了一个用户态异常，进而会==导致一系列的陷阱动作，它们都是由硬件自动完成的==：
- 将用户的特权模式==从U-Mode提升至S-Mode==，为陷阱的处理做准备。
- 将==当前正在执行的指令地址(或当前指令的下一条)放入sepc中保存==。
- 将stvec中保存的==trampoline程序入口地址==放入pc中，准备进入。
- 将当前==导致陷阱的原因记录在scause寄存器中==。
- 将==当前模式保存在sstatus的SPP位==，并==清空sstatus中的SIE位来关闭中断==，之前的SIE位保存在SPIE位。进行关中断
- 更新stval寄存器的值，使其==指向出现异常的地址==

上述步骤都是硬件流程。在RISC-V的设计中，==为了最大程度地保证指令集的可移植性==，RISC-V选择将灵活性交给软件设计者，而硬件只完成==很少的必要的工作==。

现在pc的值被设置为stvec的值了，上面的步骤都是==CPU内部的硬件电路完成的==。所以程序即将从之前stvec保存的地址值开始执行，而stvec指向了谁呢？——**uservec***

# 4.1 RISC-V trap machinery
Each RISC-V CPU has a set of control registers that the kernel writes to tell the CPU how to handle traps. `riscv.h` contains definitions that xv6 uses. Here's an outline of the most important registers:
- `stvec`: The kernel writes the address of its trap handler here; the RISC-V jumps to the address in `stvec` to handle a trap.
- `sepc`: When a trap occurs, RISC-V saves the program counter here (since the `pc` is then overwritten with the value in `stvec`). The `sret` (return from trap) instruction copies `sepc` to the `pc`.
- `scause`: RISC-V puts a number here that describes the reason for the trap.
- `sscratch`: The kernel places a value here that comes in handy at the very start of a trap handler.
- `sstatus`: sstatus 中的 SIE 位控制是否启用设备中断。如果内核清除 SIE，RISC-V 将推迟设备中断，直到内核设置 SIE。SPP 位指示陷阱是来自用户模式还是管理员模式，并控制 sret 返回到哪种模式。
上述寄存器与在管理员模式下处理的陷阱有关，它们无法在用户模式下读取或写入。对于在机器模式下处理的陷阱，有一组类似的控制寄存器；xv6 仅在定时器中断的特殊情况下使用它们。

多核芯片上的每个 CPU 都有自己的一组寄存器，并且可能在任意给定时间有多个 CPU 正在处理陷阱。

当需要强制陷阱时，RISC-V 硬件会针对所有陷阱类型（定时器中断除外）执行以下操作：
1. If the trap is a device interrupt, and the `sstatus` SIE bit is clear, don't do any of the following.
2. Disable interrupts by clearing the SIE bit in `sstatus`.
3. Copy the `pc` to `sepc`.
4. Save the current mode (user or supervisor) in the SPP bit in `sstatus`.
5. Set `scause` to reflect the trap's cause.
6. Set the mode to supervisor.
7. Copy `stvec` to the `pc`.
8. Start executing at the new `pc`.

请注意，CPU 不会切换到内核页表，不会切换到内核中的堆栈，也不会保存除 pc 之外的任何寄存器。内核软件必须执行这些任务。CPU 在陷阱期间只做很少的工作，原因之一是为软件提供灵活性；例如，某些操作系统在某些情况下会省略页表切换以提高陷阱性能。

值得考虑的是，是否可以省略上面列出的任何步骤，也许是为了寻找更快的陷阱。虽然在某些情况下，更简单的序列可以工作，但通常省略许多步骤会很危险。例如，假设 CPU 没有切换程序计数器。然后来自用户空间的陷阱可以切换到管理模式，同时仍在运行用户指令。这些用户指令可能会破坏用户/内核隔离，例如通过修改 satp 寄存器以指向允许访问所有物理内存的页表。因此，CPU 切换到内核指定的指令地址（即 stvec）非常重要。

# 4.2 Traps from user space
Xv6 handles traps differently depending on whether it is executing in the kernel or in user code. 

A trap may occur while executing in user space if the user program makes a system call(`ecall` instruction), or does something illegal, or if a device interrupts. 

A major constraint on the design of xv6's trap handling is the fact that the RISC-V hardware does not switch page tables when it forces a trap. This means that the trap handler address in `stvec` must have a valid mapping in the user page table，因为这是陷阱处理代码开始执行时有效的页表。Furthermore, xv6's trap handling code needs to switch to the kernel page table; in order to be able to continue executing after that switch, the kernel page table must also have a mapping for the handler pointed to by `stvec`.

xv6 使用 trampoline 页来满足这些要求。trampoline 页包含 uservec，即 stvec 指向的 xv6 陷阱处理代码。trampoline 页映射到每个进程的页表中的地址 TRAMPOLINE，该地址位于虚拟地址空间的末尾，因此它将位于程序自身使用的内存之上。trampoline 页还映射到内核页表中的地址 TRAMPOLINE。参见图 2.3 和图 3.3。由于 trampoline 页映射到用户页表中，因此使用 PTE_U 标志，陷阱可以在管理员模式下开始在那里执行。由于 trampoline 页映射到内核地址空间中的相同地址，因此陷阱处理程序在切换到内核页表后可以继续执行。

uservec 陷阱处理程序的代码位于 trampoline.S (kernel/trampoline.S:16) 中。当 uservec 启动时，所有 32 个寄存器都包含中断用户代码拥有的值。这 32 个值需要保存在内存中的某个位置，以便在陷阱返回到用户空间时可以恢复它们。存储到内存需要使用寄存器来保存地址，但此时没有可用的通用寄存器！幸运的是，RISC-V 以 sscratch 寄存器的形式提供了帮助。uservec 开头的 csrrw 指令交换了 a0 和 sscratch 的内容。现在用户代码的 a0 保存在 sscratch 中；uservec 有一个寄存器 (a0) 可以使用；a0 包含内核先前放置在 sscratch 中的值。

**这里思考一个问题，谁在什么时候将sscratch寄存器的值写为TRAPFRAME的？)**

`uservec`'s next task is to save the 32 user registers. Before entering user space, the kernel set `sscratch` to point to a per-process `trapframe` structure that has space to save the 32 user registers. Because `satp` still refers to the user page table, `uservec` needs the trapframe to be mapped in the user address space. When creating each process, xv6 allocates a page for the process's trapframe, and arranges for it always to be mapped at user virtual address `TRAPFRAME`, which is just below `TRAMPOLINE`. The process's `p->trapframe` also points to the trapframe, though at its physical address so the kernel can use it though the kernel page table.

可以看到一个trapframe中不仅保存了RISC-V的所有通用寄存器，还保存了包括内核栈指针、内核页表等等一系列关键信息。现在==trapframe的地址已经保存在a0寄存器里了==，是时候将现有的寄存器数据存放到trapframe帧里了，在阅读下面的代码时注意一件事情，我们这里==操纵的所有地址都是虚拟地址==，这些代码能够正确地执行，**是因为我们目前还处于用户地址空间**，一旦切换到内核地址空间，==a0这个地址就不会正确对应到trapframe了==。

Thus after swapping `a0` and `sscratch`, `a0` holds a pointer to the current process's trapframe. `uservec` now saves all user registers there, including the user's `a0`, read from `sscratch`.

The `trapframe` contains the address of the current process's kernel stack, the current CPU's hartid, the address of the `usertrap` function, and the adddress of the kernel page table. `uservec` retrieves these values, switches `satp` to the kernel page table, and calls `usertrap`.

至此，uservec完成了所有寄存器信息的保存，接下来准备==转换进入内核态==，看看它具体在做什么，首先是==设置内核栈指针==，回顾一下内核地址空间会发现，每个进程都有自己的内核栈，这个内核栈是==在系统启动时就已经分配好并映射在内核地址空间中的==(kernel/proc.c:32)，现在我们只是设置好栈指针：
![[Pasted image 20241216224542.png]]
一共最多有64个进程

接下来，将==其他切换入内核态需要的信息加载到对应寄存器==，细节我都注释在了下方的代码中，如下所示：
![[Pasted image 20241216224836.png]]

![[Pasted image 20241216225003.png]]
这是整个usertrap函数的开头，除了which_dev是一个临时变量以外，这里还加了一个异常判断，那就是读取sstatus寄存器的值。我们看一下RISC-V中==对sstatus寄存器的定义==以及一段说明：

SPP位提示了CPU在进入supervisor模式之前的运行模式，当执行一个陷阱时，SPP为0表示陷阱源于用户模式，1表示其他模式)。

![[Pasted image 20241216225059.png]]


usertrap 的工作是确定陷阱的原因，对其进行处理并返回（kernel/-trap.c:37）。它首先更改 stvec，以便内核中的陷阱将由 kernelvec 而不是 uservec 处理。它保存 sepc 寄存器（保存的用户程序计数器），因为 usertrap 可能会调用yield 切换到另一个进程的内核线程，而该进程可能会返回到用户空间，在该进程中它将修改 sepc。如果陷阱是系统调用，usertrap 会调用 syscall 来处理它；如果是设备中断，则调用 devintr；否则它是一个异常，内核会终止故障进程。系统调用路径将保存的用户程序计数器加四，因为 RISC-V 在系统调用的情况下，将程序指针指向 ecall 指令，但用户代码需要在后续指令处恢复执行。在退出时，usertrap 会检查进程是否已被终止或是否应让出 CPU（如果此陷阱是计时器中断）。

我们看到usertrap这个函数其实非常简单，它真的就是一个中转站，将来自用户态的陷阱分门别类地处理，因此我们==甚至可以将它的代码骨架抽象如下==
![[Pasted image 20241216230150.png]]

决定上述走向的就是scause中的异常码，这是==RISC-V在规范中规定好的==，8对应的就是==源自用户态的系统调用==，如下表所示：
![[Pasted image 20241216230323.png]]

好了，让我们总结一下，usertrap函数通过读取scause确定了陷阱的类型，当确定是系统调用时，将把==处理权交给syscall执行下一步动作==，现在故事到了syscall函数中。

当执行完这个系统调用完成相应的功能后，返回值将被放入trapframe的a0寄存器，这样返回以后用户态函数就可以按照calling conventions的约定从trapframe中读取对应的返回值了。==比如sys_trace、sys_sysinfo等，它们的本质就是内核功能函数==，而在usys.pl中加入的一个个entry，本质上就系统调用对应的用户态接口，它们==只负责将对应的系统调用号载入到a7寄存器==。

![[Pasted image 20241216232659.png]]

sstatus寄存器中保存着很多记录CPU信息的位，这些位里有一些是描述当前状态的，而有一些是记录上一个状态的。以这段代码中的SPP和SPIE来说，RISC-V规范中对SPP的描述是这样的：

SPP：The SPP bit indicates the privilege level at which a hart was executing before entering supervisor mode. When a trap is taken, SPP is set to 0 if the trap originated from user mode, or 1 otherwise. When an SRET instruction (see Section 3.3.2) is executed to return from the trap handler, the privilege level is set to user mode if the SPP bit is 0, or supervisor mode if the SPP bit is 1; SPP is then set to 0.(SPP位记录了一个CPU进入supervisor模式之前的特权等级，当执行一个陷阱时，如果是从用户态而来，SPP将会被设置为0，而在其它情况下会被设置为1。当SRET指令执行以准备从陷阱中返回时，如果SPP位为0，特权等级将降至用户模式，如果为1则降至supervisor模式；SPP在此之后会被置为0)。

SPIE：The SPIE bit indicates whether supervisor interrupts were enabled prior to trapping into supervisor mode. When a trap is taken into supervisor mode, SPIE is set to SIE, and SIE is set to 0. When an SRET instruction is executed, SIE is set to SPIE, then SPIE is set to 1.(SPIE位表示在陷入supervisor模式之前，supervisor模式下的中断是否开启。当一个陷阱陷入supervisor模式之后，SPIE会被设置成SIE，SIE则会被设置成0(也就是关中断)，==当SRET指令执行时，SIE会被设置成SPIE，SPIE会被设置成1)==。

The first step in returning to user space is the call to `usertrapret`. This function sets up the RISC-V control registers to prepare for a future trap from user space. This involves changing `stvec` to refer to `uservec`, preparing the trapframe fields that `uservec` relies on, and setting `sepc` to the previously saved user program counter. At the end, `usertrapret` calls `userret` on the trampoline page that is mapped in both user and kernel page tables; the reason is that assembly code in `userret` will switch page tables.

好，最后简单总结一下usertrapret做的事情：
- 关中断(直到执行SRET之前，始终不响应中断)
- 设置stvec到uservec
- 设置trapframe中与内核有关的信息，为下一次陷阱处理做准备
- 正确地设置sepc，跳转回正确的地址继续执行
- 调用userret函数，准备==恢复现场、切换页表、设置sscratch==

`usertrapret`'s call to `userret` passes `TRAPFRAME` in `a0` and a pointer to the process's user page table in `a1`. `userret` switches `satp` to the process's user page table. Recall that the user page table maps both the trampoline page and `TRAPFRAME`, but nothing else from the kernel. The fact that the trampoline page is mapped at the same virtual address in user and kernel page tables is what allows `uservec` to keep executing after changing `satp`. `userret` copies the trapframe's saved user `a0` to `sscratch` in preparation for a later swap with 
TRAPFRAME. From this point on, the only data `userret` can use is the register contents and the content of the trapframe. Next `userret` restores saved user registers from the trapframe, does a final swap of `a0` and `sscratch` to restore the user `a0` and save `TRAPFRAME` for the next trap, and executes `sret` to return to user space.

sret指令完成了以下一些事情(由硬件完成的)：

- 将sepc中保存的值放入PC，==sepc的值在usertrapret中已经被设置好了==，对于系统调用来说，它返回的是系统调用的下一条指令
- 将权限模式设置为sstatus中的SPP位，这在==usertrapret中也已经设置好了==，0表示返回用户模式
- 将==sstatus中SIE位的值设置为SPIE==，在usertrapret中SPIE被设置为1表示开启中断
- 最后将sstatus中的SPP置为0，SPIE置为1

**经过这一步，我们已经完成了整个系统调用的流程，正式地返回了用户态，并且当前PC指向了ecall指令的下一条指令。**

重返用户态，ret 尾声

经过userret的设置代码执行，现在终于回到了用户态，并且PC指向了ecall指令的下一条指令，让我们再回到sh.asm中==看看它到底执行到了哪里==：
![[Pasted image 20241216233203.png]]

# 4.3 Code: Calling system calls
Chapter 2 ended with `initcode.S` invoking the `exec` system call.

`initcode.S` places the arguments for `exec` in registers `a0` and `a1`, and puts the system call number in `a7`. System call numbers match the entries in the `syscalls` array, a table of function pointers. The `ecall` instruction traps into the kernel and causes `uservec`, `usertrap`, and then `syscall` to execute, as we saw above.

`syscall` retrieves the system call number from the saved `a7` in the trapframe and uses it to index into `syscalls`. And `a7` contains `SYS_exec`, resulting in a call to the system call implementation function `sys_exec`.

When `sys_exec` returns, `syscall` records its return value in `p->trapframe->a0`. This will cause the original user-space call to `exec()` to return the value, since the C calling convention on RISC-V places return values in `a0`. System calls conventionally return negative numbers to indicate errors, and zero or positive numbers for success. If the system call number is invalid, `syscall` prints an error and returns -1.

# 4.4 Code: System call arguments
System call implementation in the kernel need to find the arguments passed by user code: in registers. The kernel trap code saves user registers to the current process's trap frame, where kernel code can find them. The kernel functions `argint`, `argaddr`, and `argfd` retrieve the n' th system call argument from the trap frame as an integer, pointer, or a file descriptor. They all call `argraw` to retrieve the appropriate saved user register.

Some system calls pass pointers as arguments, and the kernel must use those pointers to read or write user memory. The `exec` system call, for example, passes the kernel an array of pointers referring to string arguments in user space. These pointers pose two challenges. First, the user program may be buggy or malicious, and may pass the kernel an invalid pointer or a pointer intended to trick the kernel into accessing kernel memory instead of user memory. Second, the xv6 kernel page table mappings are not the same as the user page table mappings, so the kernel cannot use ordinary instructions to load or store from user-supplied addresses.

The kernel implements functions that safely transfer data to and from user-supplied addresses. `fetchstr` is an example. File system calls such as `exec` use `fetchstr` to retrieve string file-name arguments from user space. `fetchstr` calls `copyinstr` to do the hard work.

`copyinstr` copies up to `max` bytes to `dst` from virtual address `srcva` in the user page table `pagetable`. Since `pagetable` is not the current page table, `copyinstr` uses `walkaddr` (which calls `walk`) to look up `srcva` in `pagetable`, yielding physical address `pa0`. The kernel maps each physical RAM address to the corresponding kernel virtual address, so `copyinstr` can directly copy string bytes from `pa0` to `dst.walkaddr` checks that the user-supplied virtual address is part of the process's user address space, so programs cannot trick the kernel into reading other memory. A similar function `copyout` copies data from the kernel to a user-supplied address.

# 4.5 Traps from kernel space
Xv6 configures the CPU trap registers somewhat differently depending on whether user or kernel code is executing. When the kernel is executing on a CPU, the kernel points `stvec` to the assembly code at `kernelvec`. Since xv6 is already in the kernel, `kernelvec` can rely on `satp` being set to the kernel page table, and on the stack pointer referring to a valid kernel stack. `kernelvec` pushes all 32 registers onto the stack, from which it will later restore them so that the interrupted kernel code can resume without disturbance.

kernelvec 将寄存器保存在中断的内核线程的堆栈上，这是有道理的，因为寄存器值属于该线程。如果陷阱导致切换到不同的线程，这一点尤其重要——在这种情况下，陷阱实际上将从新线程的堆栈返回，将中断线程保存的寄存器安全地留在其堆栈上。

`kernelvec` jumps to `kerneltrap` after saving registers. `kerneltrap` is prepared for two types of traps: device interrupts and exceptions. It calls `devintr` to check for and handle the former. If the trap isn't a device interrupt, it must be an exception, and that is always a fatal error if it occurs in the xv6 kernel; the kernel calls `panic` and stops executing.

当 CPU 从用户空间进入内核时，xv6 会将 CPU 的 stvec 设置为 kernelvec；您可以在 usertrap (kernel/trap.c:29) 中看到这一点。有一个时间窗口，内核已开始执行，但 stvec 仍设置为 uservec，并且至关重要的是，在此窗口期间不会发生任何设备中断。幸运的是，RISC-V 在开始采取陷阱时始终会禁用中断，并且 xv6 直到设置 stvec 后才会再次启用它们。

# 4.6 Page-fault exceptions
Xv6's response to exceptions is quite boring: if an exception happends in user space, the kernel kills the faulting process. If an exception happens in the kernel, the kernel panics. Real opearting systems often respond in much more interseting ways.

As an example, many kernels use page faults to implement `copy-on-write` fork.

The CPU raises a page-fault exception when a virtual address is used that has no mapping in the page table, or has a mapping whose `PTE_V` flag is clear, or a mapping whose permission bits(`PTE_R`, `PTE_W`, `PTE_X`, `PTE_U`) forbid the operation being attempted. RISC-V distinguishes three kinds of page fault: load page faults, store page faults, and instruciton page faults.写时复制需要
记账来帮助决定何时可以释放物理页面，因为每个页面都可以被不同数量的页表引用，具体取决于分叉、页面错误、执行和退出的历史记录。这种记账可以实现一个重要的优化：如果某个进程发生存储页面错误，并且物理页面仅从该进程的页表中引用，则不需要复制。

The combination of page tables and page faults opens up a wide range of interesting possibilities in addition to COW fork. Another widely-used feature is called `lazy allocation`, which has two parts. First, when an application asks for more memory by calling `sbrk`, the kernel notes the increase in size, but does not allocate physical memory and does not create PTEs for the new range of virtual addresses. Second, on a page fault on one of thouse new addresses, the kernel allocates a page of physical memory and maps it into the page table.

Lazy allocation incurs the extra overhead of page faults, which involve a kernel/user transition. 操作系统可以通过为每次页面错误分配一批连续的页面（而不是一页）以及为此类页面错误专门设置内核进入/退出代码来降低这种成本。

另一个广泛使用的利用页面错误的特性是 demand paging。在 exec 中，xv6 将应用程序的所有文本和数据急切地加载到内存中。由于应用程序可能很大，并且从磁盘读取的成本很高，因此用户可能会注意到此启动成本：当用户从 shell 启动大型应用程序时，用户可能需要很长时间才能看到响应。为了缩短响应时间，现代内核会为用户地址空间创建页表，但会将页面的 PTE 标记为无效。在发生页面错误时，内核会从磁盘读取页面的内容并将其映射到用户地址空间。与 COW fork 和惰性分配一样，内核可以透明地向应用程序实现此功能。

计算机上运行的程序可能需要比计算机 RAM 更多的内存。为了妥善应对，操作系统可以实现分页到磁盘。其想法是只将一小部分用户页面存储在 RAM 中，其余部分存储在磁盘的分页区域中。内核将与存储在分页区域中（因此不在 RAM 中）的内存相对应的 PTE 标记为无效。如果应用程序尝试使用已分页到磁盘的页面之一，则该应用程序将引发页面错误，并且必须将页面分页：内核陷阱处理程序将分配一个物理 RAM 页面，将页面从磁盘读入 RAM，并修改相关的 PTE 以指向 RAM。

如果需要调入某个页面，但没有可用的物理 RAM，会发生什么情况？在这种情况下，内核必须首先释放物理页面，方法是将其调出或逐出到磁盘上的分页区域，并将引用该物理页面的 PTE 标记为无效。逐出成本很高，因此，如果分页频率不高，则分页效果最佳：如果应用程序仅使用其内存页面的子集，并且子集的并集适合 RAM。此属性通常被称为具有良好的引用局部性。与许多虚拟内存技术一样，内核通常以对应用程序透明的方式实现分页到磁盘。

无论硬件提供多少 RAM，计算机通常都只能在很少甚至没有可用物理内存的情况下运行。例如，云提供商在一台机器上多路复用许多客户，以经济高效地使用其硬件。再举一个例子，用户在少量物理内存中在智能手机上运行许多应用程序。在这种情况下，分配页面可能需要首先驱逐现有页面。因此，当可用物理内存稀缺时，分配成本很高。

当可用内存稀缺时，延迟分配和按需分页尤其有利。在 sbrk 或 exec 中积极分配内存会产生额外的驱逐成本，以使内存可用。此外，还存在浪费积极工作的风险，因为在应用程序使用该页面之前，操作系统可能已经将其驱逐。

# 4.7 Real world
trampoline 和 trapframe 可能看起来过于复杂。驱动力在于 RISC-V 在强制执行陷阱时故意尽可能少地执行操作，以便实现非常快速的陷阱处理，而这非常重要。因此，内核陷阱处理程序的前几个指令必须在用户环境中执行：用户页表和用户寄存器内容。并且陷阱处理程序最初不知道有用的事实，例如正在运行的进程的身份或内核页表的地址。解决方案是可能的，因为 RISC-V 提供了受保护的位置，内核可以在进入用户空间之前将信息存放在这些位置：sscratch 寄存器和指向内核内存但因缺少 PTE_U 而受到保护的用户页表条目。xv6 的 trampoline 和 trapframe 利用了这些 RISC-V 功能。

如果将内核内存映射到每个进程的用户页表（使用适当的 PTE 权限标志），则可以消除对特殊 trampoline 页的需求。这还将消除从用户空间捕获到内核时进行页表切换的需要。这反过来又允许内核中的系统调用实现利用当前进程正在映射的用户内存，从而允许内核代码直接取消引用用户指针。许多操作系统都采用了这些想法来提高效率。xv6 避免了这些想法，以减少由于无意中使用用户指针而导致内核出现安全漏洞的可能性，并降低确保用户和内核虚拟地址不重叠所需的一些复杂性。


