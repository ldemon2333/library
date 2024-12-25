# 7.1 Multiplexing
Xv6 multiplexes by switching each CPU from one process to another in two situations.
- `sleep` and `wakeup` mechanism switches when a process waits for device or pipe I/O to complete, or waits for a child to exit, or waits in the `sleep` system call.
- xv6 periodically forces a switch to cope with processes that compute for long periods without sleeping.

How to switch from one process to another? Context switching is some of the most opaque code in xv6. 

How to force switches in a way that is transparent to user processes? Xv6 uses the standard technique in which a hardware timer's interrupts drive context switches.

Third, all of the CPUs switch among the same shared set of processes, and a locking plan is necessary to avoid races. 

Fourth, a process's memory and other resources must be freed when the process exits, but it cannot do all of this itself because it can't free its own kernel stack while still using it.

Fifth, each core of a multi-core machine must remember which process it is executing so that system calls affect the correct process's kernel state. 

Finally, `sleep` and `wakeup` allow a process to give up the CPU and wait to be woken up by another process or interrupt.

# 7.2 Code: Context switching
Figure 7.1 outlines the steps involved in switching from one user process to another: a user-kernel transition (system call or interrupt) to the old process's kernel thread, a context switch to the current CPU's scheduler thread, a context switch to a new process's kernel thread, and a trap return to the user-level process. The xv6 scheduler has a dedicated thread (saved registers and stack) per CPU because it is not safe for the scheduler execute on the old process's kernel stack.

Switching from one thread to another involves saving the old thread's CPU registers, and restoring the previously-saved registers of the new thread; the fact that the stack pointer and program counter are saved and restored means that the CPU will switch stacks and switch what code it is executing.

![[Pasted image 20241223162443.png]]
`swtch` saves only callee-saved registers; the C compiler generates code in the caller to save caller-saved registers on the stack. It does not save the program counter. Instead, `swtch` saves the `ra` register, which holds the return address from which `swtch` was called. Now `swtch` restores registers from the new context, which holds register values saved by a previous `swtch`. When `swtch` returns, it returns to the instructions pointed to by the restored `ra` register, that is, the instruction from which the new thread previously called `swtch`. In addition, it returns on the new thread's stack, since that's where the restored `sp` points.

When the `swtch` we have been tracing returns, it returns not to `sched` but to `scheduler`, with the stack pointer in the current CPU's scheduler stack.

在 **RISC-V** 架构中，**`x1`** 寄存器通常是用于存储 **返回地址**，因此它有时被称为 **`ra`**（Return Address）寄存器。

`swtch`只保存callee saved寄存器，caller saved寄存器在栈中被调用的代码保存。`swtch`并没有保存`pc`寄存器，而是保存了`ra`，当恢复了新的进程之前保存的`ra`寄存器后，将返回到`ra`寄存器指向的上一个进程调用`swtch`的代码。如果保存`pc`寄存器，将只能回到`swtch`本身。由于切换到的`&mycpu()->context`是被`scheduler`对`swtch`的调用所保存的，因此当进行`swtch`时，我们将返回到`scheduler`，栈指针也将指向当前CPU的scheduler stack。

当一个新的进程是第一次被`scheduler`调度的时候，不返回到`sched`，而是返回到`forkret`（因为之前并没有从`sched`中调用过`swtch`）。`forkret`将`p->lock`释放掉，然后回到`usertrapret`。

### `x1` 寄存器（`ra`）的作用：

- **返回地址**：在函数调用时，程序计数器（PC）会存储下一条指令的地址，并通过 **`jal`**（Jump and Link）指令跳转到目标函数。在跳转之前，`jal` 指令会将当前指令的地址（即返回地址）存储到 `x1` 寄存器中。这样，函数执行完后，可以通过 **`jalr`** 指令使用 `x1` 中的值返回到调用者的地方。

### `x1` 寄存器的特殊性：

- `x1` 作为 **`ra`** 寄存器，是 **RISC-V 函数调用约定** 中的一部分。它通常用于保存函数返回的地址，但在实际应用中，某些操作（例如嵌套的函数调用）可能会覆盖 `x1` 寄存器的值。因此，函数调用前需要保存 `x1` 的值到栈中，以防止覆盖。


# 7.3 Code: Scheduling
You can see this sequence in `yield`, `sleep`, and `exit.` `sched` double-checks some of those requirements and then checks an implication: since a lock is held, interrupts should be disabled. 

我们刚刚看到 xv6 在调用 swtch 时会持有 p->lock：swtch 的调用者必须已经持有锁，并且锁的控制权将传递给切换到的代码。这种约定对于锁来说并不常见；通常获取锁的线程也负责释放锁，这使得正确性更容易推断。对于上下文切换，有必要打破这种约定，because `p->lock` protects invariants on the process's `state` and `context` fields that are not true while executing in `swtch`. One example of a problem that could arise if `p->lock` were not held during `swtch`.

内核线程放弃其 CPU 的唯一位置是在 sched 中，并且它始终切换到 scheduler 中的相同位置，而后者（几乎）始终切换到之前调用 sched 的某个内核线程。因此，如果打印出 xv6 切换线程的行号，就会观察到以下简单模式：(kernel/proc.c:456)、(kernel/proc.c:490)、
(kernel/proc.c:456)、(kernel/proc.c:490) 等等。有意通过线程切换相互转移控制权的过程有时称为 _coroutines_；在此示例中，sched 和 scheduler 是彼此的协同程序。

There is one case when the scheduler's call to `swtch` does not end up in `sched.allocproc` sets the context `ra` register of a new process to `forkret`, so that its first `swtch` "returns" to the start of that function.

思考调度代码结构的一种方法是，它强制执行一组关于每个进程的不变量，并且当这些不变量不成立时，保持 p->lock。一个不变量是，如果进程正在运行，则计时器中断的收益必须能够安全地从进程切换出去；这意味着 CPU 寄存器必须保存进程的寄存器值（即 swtch 尚未将它们移动到上下文），并且 c->proc 必须引用进程。另一个不变量是，如果进程是可运行的，则空闲 CPU 的调度程序必须安全地运行它；这意味着 p->context 必须保存进程的寄存器（即，它们实际上不在实际寄存器中），没有 CPU 在进程的内核堆栈上执行，并且没有 CPU 的 c->proc 引用进程。请注意，在保持 p->lock 时，这些属性通常不成立。

维护上述不变量是 xv6 经常在一个线程中获取 p->lock 并在另一个线程中释放它的原因，例如在yield 中获取并在scheduler 中释放。一旦yield 开始修改正在运行的进程的状态以使其变为RUNNABLE，锁必须保持持有直到不变量恢复：最早的正确释放点是在scheduler（在其自己的堆栈上运行）清除c->proc之后。同样，一旦scheduler 开始将RUNNABLE 进程转换为RUNNING，锁就不能被释放，直到内核线程完全运行（在swtch 之后，例如在yield 中）。

# 7.4 Code: mycpu and myproc
Xv6 often needs a pointer to the current process's `proc` structure. On a uniprocessor one could have a global variable pointing to the current `proc`. This doesn't work on a multi-core machine, since each core executes a different process. The way to solve this problem is to exploit the fact that each core has its own set of registers; we can use one of those registers to help find per-core information.

`struct cpu` for each CPU, which records the process currently running on that CPU (if any), saved registers for the CPU's scheduler thread, and the count of nested spinlocks needed to manage interrupt disabling. The function `mycpu` returns a pointer to the current CPU's `struct cpu`. RISC-V numbers its CPUs, giving each a _hartid_. Xv6 ensures that each CPU's hartid is stored in that CPU's `tp` register while in the kernel. 

`mstart` sets the `tp` register early in the CPU's boot sequence, while still in machine mode. `usertrapret` saves `tp` in the trampoline page, because the user process might modify `tp`. Finally, `uservec` restores that saved `tp` when entering the kernel from user space. The compiler guarantees never to use the `tp` register.

The return values of `cpuid` and `mycpu` are fragile: if the timer were to interrupt and cause the thread to yield and then move to a different CPU, a previously returned value would no longer be correct. To avoid this problem, xv6 requires that callers disable interrupts, and only enable them after they finish using the returned `struct cpu`.

**即使中断已启用，返回值也是安全的**： 返回的 `struct proc` 指针是安全的，即使在中断已经启用的情况下。原因是：
- **进程指针不会改变**：即使发生了定时器中断，或者进程被切换到另一个 CPU 上，当前的进程指针（`struct proc` 指针）依然有效。这是因为 `myproc` 函数获取的是当前 CPU 上的进程指针，当前进程的指针会保持一致，即使进程可能在其他 CPU 上运行。
- **进程指针与 CPU 绑定**：每个 CPU 都有一个独立的进程指针，它指向该 CPU 上正在执行的进程。如果进程被调度到其他 CPU 上，`myproc` 函数依然可以正确返回当前正在执行的进程指针，因为它是绑定到具体 CPU 的。

# 7.5 Sleep and wakeup
Sleep allows a kernel thread to wait for a specific event; another thread can call wakeup to indicate that threads waiting for an event should resume. Sleep and wakeup are often called _sequence coordination_ or _conditional synchronization_ mechanisms.

Sleep and wakeup provide a relatively low-level synchronization interface. To motivate the way they work in xv6, we'll use them to build a higher-level synchronization mechanism called a _semapohore_.
![[Pasted image 20241223211517.png]]
![[Pasted image 20241223211524.png]]
The implementation above is expensive. If the producer acts rarely, the consumer will spend most of its time spinning in the `while` loop hoping for a non-zero count. The consumer's CPU could probably find more productive work than _busy waiting_ by repeatedly _polling_ `s->count`. Avoiding busy waiting requires a way for the consumer to yield the CPU and resume only after `V` increments the count.

Let's imagine a pair of calls, `sleep` and `wakeup`, that work as follows. `sleep(chan)` sleeps on the arbitrary value `chan`, called the _wait channel_. `sleep` puts the calling process to sleep, releasing the CPU for other work. `wakeup(chan)` wakes all processes sleeping on `chan`, causing their `sleep` calls to return. If no processes are waiting on `chan`, `wakeup` does nothing.
![[Pasted image 20241223212059.png]]
`P` now gives up the CPU instead of spinning, which is nice. However, it turns out not to be straightforward to design `sleep` and `wakeup` with this interface without suffering from what is known as the _lost wake-up_ problem. Suppose that `P` finds that `s->count==0` on line 212. While `P` is between lines 212 and 213, `V` runs on another CPU: it changes `s->count` to be nonzero and calls `wakeup`, which finds no processes sleeping and thus does nothing. Now `P` continues executing at line 213: it calls `sleep` and goes to sleep. This causes a problem: `P` is asleep waiting for a `V` call that has already happened. Unless we get lucky and the producer calls `V` again, the consumer will wait forever even though the count is non-zero.

An correct way to protect the invariant would be to move  the lock acquisition in `P` so that its check of the count and its call to `sleep` are atomic:
![[Pasted image 20241223213704.png]]
One might hope that this version of `P` would avoid the lost wakeup because the lock prevents `V` from executing between lines 313 and 314. It does that, but it also deadlocks: `P` holds the lock while it sleeps, so `V` will block forever waiting for the lock.

引入一个新方法，the caller must pass the _condition lock_ to `sleep` so it can release the lock after the calling process is marked as asleep and waiting on the sleep channel.
![[Pasted image 20241223214347.png]]
![[Pasted image 20241223214355.png]]
Note that we need `sleep` to atomically release `s->lock` and put the consuming process to sleep, in order to avoid lost wakeups.

# 7.6 Code: Sleep and wakeup
The basic idea is to have `sleep` mark the current process as `SLEEPING` and then call `sched` to release the CPU. Callers of `sleep` and `wakeup` can use any mutually convenient number as the channel. Xv6 often uses the address of a kernel data structure involved in the waiting.

At some point, a process will acquire the condition lock, set the condition that the sleeper is waiting for, and call `wakeup(chan)`. It's important that `wakeup` is called while holding the condition lock.

# 7.7 Code: Pipes
A more complex example that uses `sleep` and `wakeup` to synchronize producers and consumers is xv6's implementation of pipes. 

Each pipe is represented by a `struct pipe`, which contains a `lock` and a `data` buffer. The fields `nread` and `nwrite` count the total number of bytes read from and written to the buffer. The buffer wraps around: the next byte written after `buf[PIPESIZE-1]` is `buf[0]`. `buf` 采用环形缓冲区。

Let's suppose that calls to `piperead` and `pipewrite` happen simultaneously on two different CPUs. `pipewrite` begins by acquiring the pipe's lock.

The pipe code uses separate sleep channels for reader and writed (`pi->nread` and `pi->nwrite`); this might make the system more efficient in the unlikely event that there are lots of readers and writers waiting for the same pipe.

# 7.8 Code: wait, exit and kill
At the time of the child's death, the parent may already be sleeping in `wait`, or may be doing something else; The way that xv6 records the child's demise until `wait` observes it is for `exit` to put the caller into the `ZOMBIE` state, where it stays until the parent's `wait` notices it, changes the child's state to `UNUSED`, copies the child's exit status, and returns the child's process ID to the parent.

The reason is that `wait_lock` acts as the condition lock that helps ensure that the parent doesn't miss a `wakeup` from an exiting child. If it finds a child in `ZOMBIE` state, it frees that child's resources and its `proc` structure, copies the child's exit status to the address supplied to `wait` (if it is not 0), and returns the child's process ID. If `wait` finds children but none have exited, it calls `sleep` to wait for any of them to exit, then scans again. `wait` often holds two locks, `wait_lock` and some process's `np->lock`; the deadlock-avoiding order is first `wait_lock` and then `np->lock`.

`wait`中是一个无限循环，每个循环中先对所有的进程循环查找自己的子进程，当发现有子进程并且子进程的状态为`ZOMBIE`时，将子进程的退出状态`np->xstate` `copyout`到`wait`传入的用户空间的`addr`中，然后释放掉子进程占用的所有的内存空间，返回子进程的pid。如果没有发现任何`ZOMBIE`子进程，睡眠在`p`上以等待子进程`exit`时唤醒`p`。`wait_lock`是`wait`和`exit`的condition lock用来防止错过wakeup。`wait_lock`实际上就是`wait()`调用者的`p->lock`

**注意**：`wait()`先要获取调用进程的`p->lock`作为`sleep`的condition lock，然后在发现`ZOMBIE`子进程后获取子进程的`np->lock`，因此xv6中必须遵守先获取父进程的锁才能获取子进程的锁这一个规则。因此在循环查找`np->parent == p`时，不能先获取`np->lock`，因为`np`很有可能是自己的父进程，这样就违背了先获取父进程锁再获取子进程锁这个规则，可能造成死锁。

`exit` records the exit status, frees some resources, calls `reparent` to give its children to the `init` process, wakes up the parent in case it is in `wait`, marks the caller as a zombie, and pernamently yields the CPU. `exit` holds both `wait_lock` and `p->lock` during this sequence. It holds `wait_lock` because it's the condition lock for the `wakeup(p->parent)`, preventing a parent in `wait` from losing the wakeup. `exit` must hold `p->lock` for this sequence also, to prevent a parent in `wait` from seeing that the child is in state `ZOMBIE` before the child has finally called `swtch`. `exit` acquires these locks in the same order as `wait` to avoid deadlock.

`exit`关闭所有打开的文件，将自己的子进程reparent给`init`进程，因为`init`进程永远在调用`wait`，这样就可以让自己的子进程在`exit`后由`init`进行`freeproc`等后续的操作。然后获取进程锁，设置退出状态和当前状态为`ZOMBIE`，进入`scheduler`中并且不再返回。

注意：在将`p->state`设置为`ZOMBIE`之后才能释放掉`wait_lock`，否则`wait()`的进程被唤醒之后发现了`ZOMBIE`进程之后直接将其释放，此时`ZOMBIE`进程还没运行完毕。

It  may look incorrect for `exit` to wake up the parent before setting its state to `ZOMBIE`, but that is safe: although `wakeup` may cause the parent to run, the loop in `wait` cannot examine the child until the child's `p->lock` is released by `scheduler`, so `wait` can't look at the exiting process until well after `exit` has set its states to `ZOMBIE`.

`kill` lets one process request that another terminate. Thus `kill` does very little: it just sets the victim's `p->killed` and, if it is sleeping, wakes it up. Eventually the victim will enter or leave the kernel, at which point code in `usertrap` will call `exit` if `p->killed` is set. If the victim is running in user space, it will soon enter the kernel by making a system call or because the timer (or some other device) interrupts.

`exit`是让自己的程序进行退出，`kill`是让一个程序强制要求另一个程序退出。`kill`不能立刻终结另一个进程，因为另一个进程可能在执行敏感命令，因此`kill`仅仅设置了`p->killed`为1，且如果该进程在睡眠状态则将其唤醒。当被`kill`的进程进入`usertrap`之后，将会查看`p->killed`是否为1，如果为1则将调用`exit`

If the victim process is in `sleep`, `kill`'s call to `wakeup` will cause the victim to return from `sleep`. This is potentially dangerous because the condition being waiting for may not be true.
However, xv6 calls to `sleep`  are always wrapped in a `while` loop that re-tests the condition after `sleep` returns. Some calls to `sleep` also test `p->killed` in the loop, and abandon the current activity if it is set. This is only done when such abandonment would be correct. For example, the pipe read and write code returns if the killed flag is set; eventually the code will returns back to trap, which will again check `p->killed` and exit.

Some xv6 `sleep` loops do not check `p->killed` because the code is in the middle of a multistep system call that should be atomic. The virtio driver is an example: it does not check `p->killed` because a disk operation may be one of a set of writes that are all needed in order for the file system to be left in a correct state. A process that is killed while waiting for disk I/O won't exit until it completes the current system call and `usertrap` sees the killed flag.
# 7.9 Process Locking
A simple way to think about `p->lock` is that it must be held while reading or writing any of the following `struct proc` fields: `p->state`, `p->chan`, `p->killed`, `p->xstate`, and `p->pid`. These fields can be used by other processes, or by scheduler threads on other cores, so it's natural that they must be protected by a lock.

However, most uses of `p->lock` are protecting higher-level aspects of xv6's process data structures and algorithms. Here's the full set of things that `p->lock` does:
- Along with `p->state`, it prevents races in allocating `proc[]` slots for new processes.
- It conceals a process from view while it is being created or destroyed.
- It prevents a parent's `wait` from collecting a process that has set its state to `ZOMBIE` but has not yet yielded the CPU.
- It prevents another core's scheduler from deciding to run a yielding process after it sets its state to `RUNNABLE` but before it finishes `swtch`.
- It ensures that only one core's scheduler decides to run a `RUNNABLE` processes.
- It prevents a timer interrupt from causing a process to yield while it is in `swtch`.
- Along with the condition lock, it helps prevent `wakeup` from overlooking a process that is calling `sleep` but has not finished yielding the CPU.
- It prevents the victim process of `kill` from exiting and perhaps being re-allocated between `kill`'s check of `p->pid` and setting `p->killed`.
- It makes `kill`'s check and write of `p->state` atomic.

# 7.10 Real world
In addition, complex policies may lead to unintended interactions such as `prioriy inversion` and `convoys`. Priority inversion can happen when a low-priority and high-priority process both use a particular lock, which when acquired by the low-priority process can prevent the high-priority process from making progress.

The original Unix kernel's `sleep` simply disabled interrupts, which sufficed because Unix ran on a single-CPU system. The Linux kernel's `sleep` uses an explicit process queue, called a wait queue, instead of a wait channel; the queue has its own internal lock.

Scanning the entire set of processes in `wakeup` is inefficient. A better solution is to replace the `chan` in both `sleep` and `wakeup` with a data structure that holds a list of processes sleeping on that structure, such as Linux's wait queue. All of these mechanisms share the same flavour: the sleep condition is protected by some kind of lock dropped atomically during sleep.

唤醒的实现会唤醒所有在特定通道上等待的进程，并且可能有许多进程正在等待该特定通道。
操作系统将调度所有这些进程，它们将争相检查睡眠条件。以这种方式行事的进程有时被称为 _thundering herd_ ，最好避免这种情况。Most condition variables have two primitives for `wakeup: signal`, which wakes up one process, and `broadcast`, which wakes up all waiting processes.

信号量通常用于同步。计数通常对应于管道缓冲区中可用的字节数或进程拥有的僵尸子进程数。使用显式计数作为抽象的一部分可避免“丢失唤醒”问题：对已发生的唤醒次数进行显式计数。计数还可避免虚假唤醒和惊群问题。


---

在 Xv6 内核中有两种情况会导致 CPU 发生调度：
- 第一种，也是我们之前就已经见过的，**时钟中断导致的当前进程让出CPU使用权**，这是一种非常常见的调度算法，我们通常称其为==时间片轮转调度算法(Round Robin)==
- 第二种，我们在之前研究自旋锁(spinlock)的实现时曾说过，自旋锁是一种十分不经济的做法，它会阻塞式的调用Test&Set原子指令原地等待，进而造成严重的CPU资源空耗。为此Xv6引入了睡眠锁(sleeplock)，进而催生出了Sleep&Wakeup机制，在这种机制下也会发生CPU的调度。

# 1. 深入理解时钟中断导致的CPU调度(yield)
## 1.1 yield 何时触发
时钟中断是一个特殊的中断，响应过程分为两个阶段：**第一个阶段在M-Mode下首先触发一个S-Mode下的软中断，然后在第二阶段再去内核态响应这个软中断，通过devintr函数的转发和识别**，最终会在usertrap或kerneltrap函数中==根据devintr的返回值为2，进而触发yield()函数让出当前CPU的使用权==。代码骨架如下(以usertrap为例)：
![[Pasted image 20241221215708.png]]
我们假设当前有一个进程P响应了这次时钟中断，那么==它的控制流就会从上述地方会陷入到yield()函数中==，所以接下来故事就到了yield函数中。

## 1.2 yield 函数
yield函数非常简单，代码如下。首先它会获取当前进程以及锁，因为执行操作系统的平台有4+1个核心(1个moniter core和4个application core)，所以==多个核心在访问进程时也有可能产生并发的问题==，故每一个进程结构体也需要锁的保护。所以在**修改进程状态为RUNNNABLE时**(进程状态本质上==维持了一个不变量==，它对应到进程的某种状态，==修改状态但并未实际进入对应状态之前，我们应该加锁保持不变量的安全性)==，所以需要首先获取进程的锁。
![[Pasted image 20241221215828.png]]
顺带一提，Xv6中进程有6个状态，在这篇以及后面的博客中，我们==会看到这些状态之间的切换==，如果可以，我会在==最后给出这些进程状态切换的概览图==。
![[Pasted image 20241221220318.png]]
## 1.3 sched——交换旧用户进程和内核进程的上下文
### 1.3.1 进程上下文（context）
在进入分析sched之前，必须有必要对Xv6的==内核线程切换机理做一些简单且必要的说明==。我们在操作系统原理中经常说切换进程时要==保存和恢复上下文(context)==，Xv6中也做了相同的工作，有一个结构体context专门用来存放一些通用寄存器，定义如下所示(kernel/proc.h:2)：
![[Pasted image 20241221220521.png]]
这个结构体中声明的s0-s11寄存器属于==被调用者保存的寄存器(Callee-Saved Registers)==，简单插入一些背景知识，在函数调用时寄存器组中的所有寄存器可以分为**调用者保存(Caller-Saved)和被调用者保存(Callee-Saved)两种：

- **调用者保存的寄存器**，就是在函数调用发生之前，C编译器会==自动将本次函数中会用到的寄存器自动入栈保存==，等到==从函数返回后这些寄存器会自动弹栈恢复==，所以被调用者可以在函数内放心使用这些寄存器，**之所以不用在context中保存caller-saved registers，就是因为在运行swtch(下面讲到)函数时，编译器自动帮我们将它们做了保存**。
- **被调用者保存的寄存器**，则需要被调用的函数在函数开头==主动将自己会用到的函数入栈保存==，在==函数功能执行完成返回之前逐一弹栈恢复==，最后再返回。RISC-V的标准对不同寄存器的保存职责规定如下：
![[Pasted image 20241221220723.png]]
我们可以看到==s0-s11都是被调用者保存的寄存器==，我们切换到新进程之后一定会用到这些寄存器，所以context保存它们是应该的。保存堆栈指针sp是为了==可以在以后进程重新被调度时内核栈可以被成功恢复==，保存返回地址ra则是为了让==进程恢复调度时指令可以连续执行==，因为**上下文切换会使得指令流切换到另一个进程中去**，所以原有的ra寄存器的内容会被篡改，我们一会儿就会在swtch汇编代码中看到:)

所以我们可以得出这样一个简单的结论：在Xv6中，context = callee-saved registers + ra + sp，

### 1.3.2 两次交换来实现线程调度和切换——swtch函数
Xv6中是通过==多次交换的方式==来实现进程切换的，==调用一次swtch函数完成一次交换动作==，这是一段汇编写成的代码(kernel/swtch.S:8)，如下所示：

这里的代码逻辑似乎看上去很清晰，==无非是存了一批寄存器，载入了一批寄存器==。其中ra寄存器和sp寄存器是最重要的部分，==它们的切换本质上完成了控制流和内核栈的切换==。**但上述这段代码实质上涉及到三个进程上下文，一个硬件上下文，两个软件上下文(其中有一个是特殊的内核进程的上下文)。**

内核进程在Xv6中是==独立于应用进程组proc(kernel/proc.c:11)的特殊进程==，拥有==独立的、静态分配的内核栈stack0(kernel/start.c:11)==，并且和CPU内核强绑定(比如，内核进程的上下文存放在cpu结构体中，而不像普通的进程一样存放在proc结构体中)。**它在启动时就开始运行并完成本核心的初始化任务，在系统启动完成之后则作为调度进程，参与本CPU调度的中间过程，挑选下一个要调度的新进程并完成进程切换。**

转回swtch函数，首先要有一个概念，==上述我们所说的context是一个软件概念==，对应的存储空间在内存里。而==CPU硬件核心内部本身也有一个寄存器组==，而这是由硬件实现的，如果==将Xv6的整个进程切换过程绘制成更加通俗易懂的图片==，会是这样：
![[Pasted image 20241221221334.png]]
- ①：在发生调度之前，==旧进程P1的上下文正占据着CPU核心中的硬件寄存器堆，在执行着自己的逻辑==。发生调度之后，**swtch的第一个动作(对应于汇编的前半段)，就是将原本正在硬件寄存器堆中执行的上下文存回位于内存的P1上下文结构体中。**
- ②：**CPU将内核进程的上下文载入硬件寄存器堆，开始调度过程**，这==对应于swtch代码的后半段==.
- ③：调度器找到了合适的可以调度的进程，**内核进程再次被换出，硬件寄存器堆中的值被存回内核进程上下文**，这对应于==第二次调用swtch时前半段代码做的事情==。
- ④：**新进程上下文被载入硬件寄存器堆**，这==对应于第二次调用swtch时后半段代码==，自此核心开始在新进程上运行。
所以，Xv6中通过上述的两次交换过程(两次swtch)成功实现了进程的调度和切换，下面我们==从代码的角度形式仔细研究一下上图中涉及到的细节==。

### 1.3.3 sched 函数的第一次switch
sched函数的代码逻辑还是非常简单的，==首先它做了一系列的合法性检查==，随后==它记录了当前CPU在进行调度之前的原始中断状态以备返回时恢复==，随后调用swtch函数，**成功将控制流切换到内核线程，在那里将进行线程的选择和调度**。

## 1.4 scheduler函数（第二次switch）
我们的==控制流成功切换到了内核进程中，那么内核进程将会从哪里开始执行呢==？这里我们先给出一个结论，从scheduler函数中的一个地方开始，你可能会问，**内核进程的代码是怎么会在scheduler函数中等着呢？

![[Pasted image 20241221222614.png]]
简单来说==这个函数实现了CPU的调度==，但是调度算法非常的朴素，就是**一次次遍历进程组，找出其中第一个满足RUNNABLE状态的进程开始调度**。每一次循环都会接着上一次结束的位置向后扫描，这样保证了==相对公平性==，否则==ID号小的进程就会被持续调度==。

除此之外有两个难点需要我们注意，如下：

### 1.4.1 注意点1——scheduler与yield中的跨进程锁机制
==scheduler函数和yield函数中出现了不同寻常的锁机制==，它**首先由一个进程获取锁，而由另外一个进程释放**。将这个过程画成图片效果如下：
![[Pasted image 20241221223441.png]]
旧进程在让出当前CPU时，首先要获取进程的锁，防止并发错误。然后==通过sched函数会调度到内核进程==，内核进程==恢复的地方正是上一次退出的地方==(即：c->proc=0是即将执行的指令)。接下来内核进程的操作就是释放锁(release(&p->lock))，这个**所谓的p正是之前在CPU上运行的旧进程。**

同样的，scheduler()在==尝试调度一个新进程之前也会先获取它的锁==，这把锁在返回新进程执行时，将会在==新进程返回到yield()函数处时释放这把由内核进程加上的锁==，这种跨进程的锁机制使用非常少见，这里值得引起我们的重视。

### 1.4.2 注意点2——scheduler函数中的开中断动作
注意内核线程并不一定一直在scheduler中执行，因为它开了中断。==开中断是非常有必要的行为，否则可能会引起死锁==。值得注意的地方在于，==开中断也会响应时钟中断，时钟中断可能会让出当前CPU==。

在内核陷阱kerneltrap中，是如下这样处理的(kernel/trap.c:152)，它会==使得内核进程不响应时钟中断，进而很好地规避了这个问题==：
![[Pasted image 20241221224438.png]]

## 1.5 概览——时钟中断导致的CPU调度
![[Pasted image 20241221224459.png]]

# 2. 回溯——故事的开始
上面我们卖了一个关子，那就是内核进程为何会一直在scheduler函数中等待。这是所有问题的源头，事实上这部分代码在main.c中可以相对容易地看见。我们转入文件kenrel/main.c，==执行main函数的进程都是内核进程==，我们可以在下面的代码中看到，**monitor core和application core在最后都会转入scheduler函数**。所以后面我们因为时钟中断而让出CPU时，==内核进程会一直在scheduler函数中等待着我们。==至此，整个故事就完全完整了。
![[Pasted image 20241221224644.png]]

# 1. Xv6 中的进程组和进程
大体而言，Xv6中进程的特征数据可分为私有数据和共有数据两大类，==私有属性的访问无需锁机制保护，而公有属性在访问之前必须先获取进程锁，==以防止并发带来的错误风险。Xv6中的进程可以分为==作为普通程序载体的通用进程组==和==负责初始化和进程调度任务的调度者进程==，调度者进程的上下文直接存放在CPU结构体中。而通用进程组则以全局进程组的形式存在(kernel/proc.c:11)：
![[Pasted image 20241223221603.png]]

# 2. 全局进程组的初始化
在xv6 启动时，每个 CPU 核心均会进入 main 函数，并在此完成一些内核数据结构的初始化，
![[Pasted image 20241223222033.png]]
上述代码中kvminit函数有这样一条调用链(kvminit -> kvmmake -> proc_mapstacks)，在==proc_mapstacks函数(kernel/proc.c:32)==中**为每一个进程都分配了一个物理内存页面作为进程的内核栈，并在内核页表中建立了映射关系**。

接下来，函数procinit(kernel/proc.c:46)来专门负责对==上述的全局进程组进行初始化==，它==初始化了每个进程的保护锁，并将自身内核栈的地址记录到了p->kstack字段==，以下是procinit函数的代码和注释：
![[Pasted image 20241223222058.png]]

上述在初始化进程组中的每个进程的内核栈时，使用了KSTACK宏，这个宏定义如下：
![[Pasted image 20241223222133.png]]

这与Xv6 book中对于内核地址空间的定义是一致的，这里只当做一个简单的回顾：
![[Pasted image 20241223222146.png]]
由于Xv6中的进程组proc(kernel/proc.c:11)是以**全局数组**的形式给出的，因此结构体的各字段==应当被编译器自动初始化为了0值(zero-value)==，所以**进程的初始状态应当是0**，而Xv6对于进程的6个状态的定义如下(kernel/proc.h:83)，可以看到==0对应于UNUSED状态==：
![[Pasted image 20241223222228.png]]
至此，进程组完成了初始化，可以看到现在的==进程组还是一个空空的“躯壳”，里面没有实际的程序代码和数据==，它在等待着程序的载入，那样才会为它注入灵魂，这个**空空的躯壳对应的状态就是UNUSED**。

# 3. 新进程的产生：fork 系统调用
![[Pasted image 20241223222340.png]]

## 3.1 第一个进程
Xv6中的1号进程由==monitor core启动==，对应代码位于函数userinit中(kernel/main.c:31)，userinit会将一段**二进制指令硬编码initcode**(kernel/proc.c:214)复制到1号进程的虚拟地址空间中，并将指令计数器设置为0。1号进程在返回用户地址空间之后会执行这段二进制代码，它会调用init.o这个可执行文件，进而==初始化命令行解释器(user/sh.c)等待用户输入命令行命令==。Xv6内核**解析并执行用户输入的命令时会使用fork系统调用产生新的进程，并执行(exec)命令对应的可执行程序。**

## 3.2 sys_fork 函数
![[Pasted image 20241223222857.png]]

## 3.3 proc.c/fork 函数
顺藤摸瓜，接下来，看看内核中的==fork函数(kernel/proc.c:272)==的实现，这里将**首先从宏观的角度来对fork函数的作用进行说明**，随后逐一地深入此函数调用的其他函数。fork函数的源码和对应的注释如下所示：

## 3.4 allocproc 函数——在进程组中寻找可用的空闲进程位置
fork函数中调用了allocproc函数来寻找一个==空的进程“躯壳”==，allocproc函数(kernel/proc.c:104)的**代码实现和注释**如下。allocproc主要负责==在全局进程组中寻找合适的位置来安排新的进程，一旦发现了可用的位置，则为其分配一个物理页作为trapframe页面==。**被选中的进程状态将会被切换为USED，表明此位置已经被抢占。**

## 3.5 allocpid 函数——为新进程分配进程号
allocproc函数中调用了allocpid函数用来为新的进程==分配一个进程号==，由于管理进程号的变量nextpid(kernel/proc.c:15)是一个全局变量，它记录着**下一个可用的进程号**，所以需要使用锁来保护。代码逻辑相对简单，

## 3.6 forkret 函数——新进程首次从内核态返回到用户态
在allocproc函数创建一个新的进程时，会将其上下文中的==ra寄存器的值设置为forkret函数(kernel/proc.c:141)，这意味着当新进程被调度时，会从forkret函数开始执行==。forkret函数(kernel/proc.c:507)的代码实现和注释如下，它是新进程上CPU执行的第一个函数：

最终，fork系统调用创建的新进程将会返回到用户态，==和创建它的父进程一样，fork出的子进程将会从系统调用指令ecall的下一条指令开始执行==。

# 4. 进程的自然退出——exit系统调用
在Xv6操作系统中，**必须在用户态下显式调用exit()函数才可以结束一个进程**，这一点在之前完成实验一时实验指导书已经给过我们提醒，这篇[StackOverflow的问答](https://stackoverflow.com/questions/71583679/why-return-does-not-exit-a-process-in-xv6)==进一步地讨论了在Xv6内核中为何要使用exit代替return的原因==，有兴趣可以扩展阅读一下。(**简单来说，Xv6中的新进程是通过fork创建，并通过exec系统调用载入程序镜像的，并不存在更上一层的进程来调用一个新的进程，所以return是没有意义的。**)


