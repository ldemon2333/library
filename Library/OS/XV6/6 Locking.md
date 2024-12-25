大多数内核（包括 xv6）都会交错执行多个活动。交错的一个来源是多处理器硬件：具有多个独立执行的 CPU 的计算机，例如 xv6 的 RISC-V。这些多个 CPU 共享物理 RAM，xv6 利用共享来维护所有 CPU 读写的数据结构。这种共享增加了一个 CPU 读取数据结构而另一个 CPU 正在更新数据结构的可能性，甚至多个 CPU 同时更新相同的数据；如果没有精心设计，这种并行访问可能会产生不正确的结果或损坏的数据结构。即使在单处理器上，内核也可能会在多个线程之间切换 CPU，导致它们的执行交错​​。最后，如果中断发生在错误的时间，修改与某些可中断代码相同数据的设备中断处理程序可能会损坏数据。并发这个词指的是由于多处理器并行、线程切换或中断而导致多个指令流交错的情况。

A lock provides mutual exclusion, ensuring that only one CPU at a time can hold the lock. Although locks are an easy-to-understand concurrency control mechanism, the downside of locks is that they can kill performance, because they serialize concurrent operations.

The rest of this chapter explains why xv6 needs locks, how xv6 implements them, and how it uses them.

# 6.1 Race conditions
As an example of why we need locks, consider two processes calling `wait` on two different CPUs. `wait` frees the child's memory. Thus on each CPU, the kernel will call `kfree` to free the children's pages. For best performance, we might hope that the `kfree` of the two parent processes would execute in parallel without either having to wait for the other, but this would not be correct given xv6's `kfree` implementation.

A race condition is a situation in which a memory location is accessed concurrently, and at least one access is a write. A race is often a sign of a bug, either a lost update (if the accesses are writes) or a read of an imcompletely-updated data structure.

We say that multiple processes `conflict` if they want the same lock at the same time, or that the lock experiences `contention`. A major challenge in kernel design is to avoid lock contention. Xv6 does little of that, but sophisticated kernels organize data structures and algorithms specifically to avoid lock contention.
![[Pasted image 20241221193547.png]]

# 6.2 Code: Locks
xv6有两种锁：spinlock和sleep-lock。spinlock的代码位于kernel/spinlock.h的`struct spinlock`中

![[Pasted image 20241221195055.png]]
`locked`为0时说明这个锁是可以acquire的。

Logically, xv6 should acquire a lock by executing code like:
![[Pasted image 20241221195300.png]]
![[Pasted image 20241221195341.png]]

这样的逻辑原子化，否则当两个不同的进程同时执行到上面的判断条件时，可能会同时获取这个锁。RISC-V是通过`amoswap r, a`来实现的，它将`a`内存地址中的内容和`r`寄存器中的内容互换，然后读取`a`内存地址的值。`amoswap` reads the value at the memory address `a`, writes the contents of register `r` to that address, and puts the value it read into `r`. 

在`acquire`中，通过一个对`amoswap`的包装函数`__sync_lock_test_and_set(&lk->locked, 1)`来实现这个原子操作，这个函数的返回值是`lk->locked`的旧的值(被换下来的值)
![[Pasted image 20241221195925.png]]
通过while不断尝试将1和`&lk->locked`互换(spinning)，当原先的`lk->locked`是0时跳出循环，这个锁被取得，否则当原先的`lk->locked`是1时不会跳出循环，并且`lk->locked`和1互换还是1，不会改变它的状态。

Once the lock is acquired, `acquire` records, for debugging, the CPU that acquired the lock. The `lk->cpu` field is protected by the lock and must only be changed while holding the lock.
![[Pasted image 20241221200345.png]]

The function `release` is the opposite of `acquire`: it clears the `lk->cpu` field and then releases the lock. Conceptually, the release just requires assigning zero to `lk->locked`. 
![[Pasted image 20241221200436.png]]

`release`是`acquire`的反向操作，先将`lk->cpu`清零。然后调用
![[Pasted image 20241221200014.png]]
将`lk->locked`置0，这也是一个原子操作。

由于编译器有时候为了性能优化会重新排列代码的执行顺序，对于顺序执行的代码来说，这种重新排列顺序并不会改变代码执行的结果，但是对于并发执行的代码，则可能改变结果，因此需要在`acquire`和`release`中用`__sync_synchronize()`来保证CPU和编译器不进行重新排列顺序。`__sync_synchronize()`是一个barrier，任何在这一行代码之前的代码都不能reorder到这一行代码的后面。

# 6.3 Code: Using locks
Second, remember that locks protect invariants: if an invariant involves multiple memory locations, typically all of them need to be protected by a single lock to ensure the invariant is maintained.

上述规则说明了何时需要锁定，但没有说明何时不需要锁定，并且为了提高效率，不要锁定太多，因为锁定会降低并行性。如果并行性不重要，则可以安排只有一个线程，而不必担心锁定。一个简单的内核可以在多处理器上做到这一点，方法是拥有一个必须在进入内核时获取并在退出内核时释放的锁（尽管管道读取或等待等系统调用会带来问题）。许多单处理器操作系统已转换为使用这种方法在多处理器上运行，有时称为“大内核锁”，但这种方法牺牲了并行性：一次只能有一个 CPU 在内核中执行。如果内核进行任何繁重的计算，使用一组更大、更细粒度的锁会更有效率，这样内核就可以同时在多个 CPU 上执行。

As an example of fine-grained locking, xv6 has a separate lock for each file, so that processes that manipulate different files can often proceed without for each other's locks. 如果希望允许进程同时写入同一文件的不同区域，则可以使文件锁定方案更加细粒度。最终，锁定粒度决策需要由性能测量和复杂性考虑决定。
![[Pasted image 20241221201845.png]]

# 6.4 Deadlock and lock ordering
如果一块代码需要同时拥有多个锁，那么应该让其他需要相同锁的进程按照相同的顺序acquire这些锁，否则可能出现死锁。比如进程1和2都需要锁A和锁B，如果进程1先acquire了锁A，进程2acquire了锁B，那么接下来进程1需要acquire锁B，进程2需要acquire锁A，但是这两个都不能acquire到也无法release各自的锁，就会出现死锁。

由于`sleep`在xv6中的机制，xv6中有很多长度为2的lock-order。比如`consoleintr`中要求先获得`cons.lock`，当整行输入完毕之后再唤醒等待输入的进程，这需要获得睡眠进程的锁。xv6的文件系统中有一个很长的lock chain，如果要创建一个文件需要同时拥有文件夹的锁、新文件的inode的锁、磁盘块缓冲区的锁、磁盘驱动器的`vdisk_lock`的锁以及调用进程的`p->lock`的锁

除了lock ordering之外，锁和中断的交互也可能造成死锁。比如当`sys_sleep`拥有`tickslock`时，发生定时器中断，定时器中断的handler也需要acquire`tickslock`，就会等待`sys_sleep`释放，但是因为在中断里面，只要不从中断返回`sys_sleep`就永远无法释放，因此造成了死锁。对这种死锁的解决方法是：==如果一个中断中需要获取某个特定的spinlock，那么当CPU获得了这个spinlock之后，该中断必须被禁用。xv6的机制则更加保守：当CPU获取了任意一个lock之后，将disable掉这个CPU上的所有中断（其他CPU的中断保持原样）。==当CPU不再拥有spinlock时，将通过`pop_off`重新使能中断.

# 6.5 Re-entrant locks
It might appear that some deadlocks and lock-ordering challenges could be avoided by using _reentrant locks_, which are also called _recursive locks_. The idea is that if the lock is held by a process and if that process attempts to acquire the lock again, then the kernel could just allow this (since the process already has the lock), instead of calling panic, as the xv6 kernel does.

但事实证明，可重入锁使得并发性推理变得更加困难：可重入锁破坏了锁导致临界区相对于其他临界区具有原子性的直觉。 Consider the following two functions f and g:
![[Pasted image 20241221203026.png]]
The intuition is that `call_once` will be called only once: either by `f`, or by `g`, but not by both.

But if re-entrant locks are allowed, and `h` happens to call `g`, `call_once` will be called _twice_.

如果不允许重入锁，那么 h 调用 g 会导致死锁，这也不太好。但是，假设调用 call_once 会是一个严重的错误，那么死锁是更好的选择。内核开发人员将观察到死锁（内核崩溃）并可以修复代码以避免它，而调用两次 call_once 可能会悄无声息地导致难以追踪的错误。

因此，xv6 使用更易于理解的非重入锁。但是，只要程序员牢记锁定规则，这两种方法都可以工作。如果 xv6 要使用重入锁，则必须修改 acquire 以注意到锁当前由调用线程持有。还必须将嵌套获取的计数添加到 struct spinlock，类似于下面讨论的 push_off。

# 6.6 Locks and interrupt handlers
一些 xv6 自旋锁保护线程和中断处理程序使用的数据。例如，clockintr 定时器中断处理程序可能会在内核线程读取 sys_sleep 中的 `ticks`（kernel/sysproc.c:64）的同时增`ticks`（kernel/trap.c:163）。锁 tickslock 将这两个访问序列化。

The interaction of spinlocks and interrupts raises a potential danger. Suppose `sys_sleep` holds `tickslock`, and its CPU is interrupted by a timer interrupt. `clockintr` would try to acquire `tickslock`, see it was held, and wait for it to be released. In this situation, `tickslock` will never be released: only `sys_sleep` can release it, but `sys_sleep` will not continue running until `clockintr` returns. So the CPU will deadlock.

为了避免这种情况，如果中断处理程序使用自旋锁，则 CPU 会启用中断。xv6 更为保守：当 CPU 获取任何锁时，xv6始终会禁用该 CPU 上的中断。中断仍可能发生在其他 CPU 上，因此中断的获取可以等待线程释放自旋锁；只是不能在同一个 CPU 上。

Xv6 re-enables interrupts when a CPU holds no spinlocks; 它必须做一些记录工作来
处理嵌套的临界区。`acquire` calls `push_off` and `release` calls `pop_off` to track the nesting level of locks on the current CPU. 当该计数达到零时，pop_off 会恢复最外层关键部分开始时存在的中断启用状态。intr_off 和 intr_on 函数分别执行 RISC-V 指令以禁用和启用中断。

It is important that `acquire` call `push_off` strictly before setting `lk->locked`. If the two were reversed, there would be a brief window when the lock was held with interrupts enabled, and an unfortunately timed interrupt would deadlock the system.

# 6.7 Instruction and memory ordering
很自然地，程序会按照源代码语句出现的顺序执行。然而，许多编译器和 CPU 会乱序执行代码，以实现更高的性能。如果一条指令需要很多个周期才能完成，CPU 可能会提前发出该指令，以便它能够与其他指令重叠，避免 CPU 停顿。例如，CPU 可能会注意到，在一系列指令中，A 和 B 并不相互依赖。CPU 可能会首先启动指令 B，要么是因为它的输入在 A 的输入之前就绪，要么是为了重叠执行 A 和 B。编译器可以通过在源代码中先发出一条语句的指令，然后再发出该语句之前的语句的指令，来执行类似的重新排序。

编译器和 CPU 在重新排序时遵循规则，以确保它们不会改变正确编写的串行代码的结果。但是，规则确实允许重新排序，从而改变并发代码的结果，并且很容易导致多处理器上的不正确行为 [2, 3]。CPU's ordering rules are called the _memory model_.

For example, in this code for `push`, it would be a disaster if the compiler or CPU moved the store corresponding to line 4 to a point after the `release` on line 6:
![[Pasted image 20241221210440.png]]
If such a re-ordering occurred, there would be a window during which another CPU could acquire the lock and observe the updated `list`, but see an uninitialized `list->next`.

To tell the hardware and compiler not to perform such re-orderings, xv6 uses `__sync_synchronize()` in both `acquire` and `release`. `__sync_synchronize()` is a _memory barrier_: it tells the compiler and CPU to not reorder loads or stores across the barrier.

# 6.8 Sleep locks
spinlock的两个缺点：1. 如果一个进程拥有一个锁很长时间，另外一个企图acquire的进程将一直等待。2. 当一个进程拥有锁的时候，不允许把当前使用的CPU资源切换给其他线程，否则可能导致第二个线程也acquire这个锁，然后自旋，无法切回到原来的线程，从而无法release锁，从而导致死锁。Yielding while holding a spinlock is illegal because it might lead to deadlock if a second thread then tried to acquire the spinlock; since `acquire` doesn't yield the CPU, the second thread's spinning might prevent the first thread from running and releasing the lock. 持有锁时 yielding 也会违反持有自旋锁时必须关闭中断的要求。Thus we'd like a type of lock that yields the CPU while waiting to acquire, and allow yields (and interrupts) while the lock is held.


xv6提供了一种 _sleep-locks_，可以在试图`acquire`一个被占有的锁时`yield` CPU。spin-lock适合短时间的关键步骤，sleep-lock适合长时间的锁。 At a high level, a sleep-lock has a `locked` field that is protected by a spinlock, and `acquiresleep`'s call to `sleep` atomically yields the CPU and releases the spinlock. The result is that other threads can execute while `acquiresleep` waits.

Because sleep-locks leave interr Beupts enabled, they cannot be used in interrupt handlers. Because `acquiresleep` may yield the CPU, sleep-locks cannot be used inside spinlock critical sections (though spinlocks can be used inside sleep-lock critical sections).

Spin-locks are best suited to short critical sections, since waiting for them wastes CPU time; sleep-locks well for lengthy operations.




# 6.9 Real world
尽管多年来一直在研究并发原语和并行性，但使用锁进行编程仍然具有挑战性。通常，最好将锁隐藏在同步队列等高级构造中，尽管 xv6 不会这样做。如果您使用锁进行编程，最好使用尝试识别竞争条件的工具，因为很容易错过需要锁的不变量。

大多数操作系统都支持 POSIX 线程 (Pthreads)，这允许用户进程在不同的 CPU 上同时运行多个线程。Pthreads 支持用户级锁、屏障等。Pthread 还允许程序员选择性地指定锁应该是可重入的。

在用户级别支持 Pthreads 需要操作系统的支持。例如，如果一个 pthread 在系统调用中阻塞，则同一进程的另一个 pthread 应该能够在该 CPU 上运行。再举一个例子，如果 pthread 更改了其进程的地址空间（例如，映射或取消映射内存），则内核必须安排运行同一进程线程的其他 CPU 更新其硬件页表以反映地址空间的变化。

可以在不使用原子指令的情况下实现锁定 [9]，但这样做成本高昂，而且大多数操作系统都使用原子指令。

如果许多 CPU 试图同时获取同一锁，锁的成本可能会很高。如果一个 CPU 在其本地缓存中缓存了一个锁，而另一个 CPU 必须获取该锁，则更新持有该锁的缓存行的原子指令必须将该行从一个 CPU 的缓存移动到另一个 CPU 的缓存，并且可能使该缓存行的任何其他副本无效。从另一个 CPU 的缓存中获取缓存行的成本可能比从本地缓存中获取缓存行的成本高出几个数量级。

为了避免与锁相关的开销，许多操作系统使用无锁数据结构和算法 [5, 11]。例如，可以实现像本章开头那样的链表，在列表搜索期间不需要锁，并且只需要一个原子指令即可将项目插入列表中。但是，无锁编程比编程锁更复杂；例如，必须担心指令和内存重新排序。使用锁进行编程已经很困难了，因此 xv6 避免了无锁编程的额外复杂性。


# 0. briefly speaking
在 Xv6 运行的平台 SiFive_Unleashed 上，也有5个核心(**1个S51 Moniter Core和4个U54 Application Core**)，这说明在Xv6系统运行的过程中==考虑并发控制(Concurrency Control)是非常必要的==。锁机制是一种==最常见的管理并发程序正确性==的方案，它通过将==对临界区的访问串行化(serialize)==来保证并发操作的正确性。Xv6中在如下地方使用了锁机制，可以作为研究对象自己去一一体悟锁的重要性(比如画画这些情况下的锁链)：

主要研究 Xv6 内核中自旋锁的实现，几个概况：
- **死锁与锁定序**：如果一个代码路径可能会在同一时间持有多个锁，那么所有代码路径都应该按照相同的次序获取这些锁，否则就会有死锁的风险。不过这个很难，==尤其是锁链非常长的时候，保持所有进程持有锁的顺序是非常艰难的。==
- **锁与中断处理程序**：有些时候==中断处理程序(interrupt handler)需要锁来保证并发安全==，那么在中断处理程序中它会尝试获取这把锁，如果进程==恰好在进入中断处理程序之前持有这把锁==，**Xv6在申请锁之前，必须先关中断。**
- **指令与内存定序：**==加入一些内存屏障指令(memory barrier)，比如fence指令，来避免这些重排序对并发造成的影响==。

# 1. 自旋锁的结构与初始化
首先来看自旋锁结构体中有什么，自旋锁结构体定义在spinlock.h中，示意如下：
![[Pasted image 20241221213218.png]]
我们可以看到，一个自旋锁的实现非常简单，其实==只需要一个标志来记录锁当前是否被占用==即可，另外附加的一些信息是用来debug时提示的。

同样的，对自旋锁的初始化过程也非常简单，代码位于kernel/spinlock.c:11，它只给锁起了个名字，并将==锁是否被持有以及当前持有锁的CPU标志==置为空，代码如下：
![[Pasted image 20241221213238.png]]

# 2. 关中断——push/pop off
Xv6为了避免这种情况采取了一种**保守的策略：在一个进程尝试获取锁（还不一定成功呢…）之前就把中断关掉。**

==push off和pop off==，它们是配对使用的。你可能会有疑问，==为什么要提供这么一套函数来控制中断的开关，我们明明使用两个宏intr_on和intr_off==(kernel/riscv.h:273, 280)就可以实现这些功能了呀?

这是因为在处理并发的过程中，我们可能会在一条代码路径上**多次获取和释放不同的自旋锁**(==这些锁按照获取的先后次序排成了一个锁链==)，又因为Xv6内核采取了上述的保守策略，在尝试获取锁之前就把中断关闭，**这样一来开关中断的动作也就随着加锁、解锁动作形成多次嵌套**，所以开关中断的动作==一定要随着加锁解锁操作配对起来==，这就需要我们额外记录一些信息。

为了提供这种支持，在保存处理器信息的结构体cpu(kernel/proc.h:22)中添加了==两个变量noff和intena来分别记录这种嵌套操作的深度==，和==进入锁链之前CPU的中断开关情况==，一旦身处锁链之中，中断一定保证是关闭的，但是最终进程一定要逐层从锁链中退出，并且**恢复到进入锁链之前的中断状态。**
![[Pasted image 20241221213559.png]]

现在就来看看分别它们的实现，首先是push_off函数，它主要做的事情就是：==关中断、递增中断动作嵌套深度==，如果当前是在尝试==获取锁链中的第一把锁==，则==将进入锁链之前的中断状态保存到intena变量中==。
![[Pasted image 20241221213929.png]]
然后就是pop_off函数，它**几乎是push_off函数的逆动作**，唯一加入的就是==一些异常状态的检测==，
![[Pasted image 20241221214202.png]]
# 3. 加锁和解锁操作
## 3.1 加锁动作——acquire
当一个进程想要对共享内存做操作且要保证正确性时，它要==尝试去争夺保护这块内存区域的锁==。这个操作我们简单称为加锁，在Xv6中它对应的函数叫做acquire(kernel/spinlock.c:21)，代码如下：

加锁代码中涉及到一些编译器的内置函数，比如==__sync_lock_test_and_set和__sync_synchronize==，这些都是==编译器对开发者管理访存模型时提供的一些工具函数==

值得注意的是__sync_lock_test_and_set在翻译为汇编指令时会有一条amoswap.w.aq指令， 这条指令**除了原子语义之外(atomic semantic)，其实还兼具内存屏障的语义(barrier semantic)**。aq标志位的使用==保证了在这条汇编之后的访存指令不会被移到这条指令的前面==，其实这和下面的fence指令语义有所重复，详见[RISC-V原子指令](https://tinylab.org/riscv-atomics/)这篇博客。而fence指令==既不允许位于本条指令之前的访存指令移到后面，也不允许位于本条指令之后的指令移到前面==，**约束是双向的**，就像是一个屏障(fence)，==牢牢将界限划分开来==。

## 3.2 解锁动作——release
解锁操作是和加锁操作对应的，用于在锁链中释放一个锁，它的代码和注释如下：

**请多注意这些原子访存指令和内存屏障指令，它们是构成锁机制的核心。**
