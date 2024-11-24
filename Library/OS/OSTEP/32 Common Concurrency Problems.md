并发bug，死锁，非死锁bugs。如何处理常见的并发bugs？

# 32.1 What Types Of Bugs Exist?
![[Pasted image 20241031151138.png]]

# 32.2 Non-Deadlock Bugs
两种非死锁bug：atomicity violation bugs and order violation bugs

## Atomicity-Violation Bugs
![[Pasted image 20241031151423.png]]

定义：“The desired serializability among multiple memory accesses is violated (i.e. a code region is intended to be atomic, but the atomicity is not enforced during execution).”

线程1 具有原子性假设，check和fputs应该是原子性的。这种问题解决是很简单的，对临界区加一个大锁。

![[Pasted image 20241031151753.png]]

## Order-Violation Bugs
![[Pasted image 20241031151958.png]]

顺序要求，如果线程2先运行，就会crash，因为mThread未定义。该问题的定义是“The desired order between two (groups of) memory accesses is flipped (i.e., A should always be executed before B, but the order is not enforced during execution)”。

使用 条件变量 来解决这种同步问题。
![[Pasted image 20241031152317.png]]

# 32.3 Deadlock Bugs
## Why Do Deadlocks Occur?
- 复杂的依赖关系
- 程序的封装性
## Conditions for Deadlock
死锁产生的必要条件：
- Mutual exclusion: Threads claim exclusive control of resources that they require (e.g., a thread grabs a lock). 
- Hold-and-wait: Threads hold resources allocated to them (e.g., locks that they have already acquired) while waiting for additional resources (e.g., locks that they wish to acquire). 
- No preemption: Resources (e.g., locks) cannot be forcibly removed from threads that are holding them. 
- Circular wait: There exists a circular chain of threads such that each thread holds one or more resources (e.g., locks) that are being requested by the next thread in the chain.

四者有一个不满足，死锁就不会产生。

## Prevention
### Circular Wait
最有用的是重写你的locking 代码，不会产生循环等待。最直接的方法是提供一个顺序在锁的获取上。

Thus, a partial ordering can be a useful way to structure lock acquisition so as to avoid deadlock.An excellent real example of partial lock ordering can be seen in the memory mapping code in Linux；

![[Pasted image 20241031223411.png]]
一种简单而有效的解决机制，顺序获得锁机制。
### Hold-and-wait
一次获取所有锁，并且立即原子执行
![[Pasted image 20241031154831.png]]

首先获取大锁 prevention，代码就保证了不会有不合时宜的线程切换。这样即使有线程以不同顺序获取L1和L2，也不会有出现 Circular Wait 导致死锁现象。

请注意，由于多种原因，该解决方案存在问题。与之前一样，封装对我们不利：在调用例程时，这种方法要求我们确切知道必须持有哪些锁并提前获取它们。这种技术还可能降低并发性，因为必须尽早（一次）获取所有锁，而不是在真正需要时获取它们。

### No Preemption，禁止抢占
设计一种更加灵活的机制。具体地，pthread_mutex_trylock() either grabs the lock (if it is available) and returns success or returns an error code indicating the lock is held; in the latter case, you can try again later if you want to grab that lock.

这种设计就可以构建一个deadlock-free，ordering-robust lock 获取协议：

![[Pasted image 20241031155659.png]]

假设有另一个线程想要获取锁，显示L2和L1，这个线程就可以主动释放L1锁，避免循环等待死锁。但这种方法会产生一个新问题：livelock。

有可能（尽管可能性不大）两个线程都反复尝试这个序列，并反复无法获得两个锁。在这种情况下，两个系统都在一遍又一遍地运行这个代码序列（因此这不是死锁），但没有取得进展，因此得名 livelock 。活锁问题也有解决方案：例如，可以在循环回并重新尝试整个操作之前添加一个随机延迟，从而降低竞争线程之间重复干扰的几率。

关于此解决方案的一点是：它绕过了使用 trylock 方法的难点。由于封装，可能再次出现的第一个问题：如果其中一个锁隐藏在正在调用的某个例程中，则跳回开头的实现会变得更加复杂。如果代码在此过程中获取了一些资源（L1 除外），则必须确保小心地释放它们；例如，如果在获取 L1 之后，代码分配了一些内存，则在获取 L2 失败时，它必须在跳回顶部再次尝试整个序列之前释放该内存。但是，在有限的情况下，这种方法可以很好地发挥作用。

您可能还会注意到，这种方法实际上并没有添加抢占，而是使用 trylock 方法允许开发人员以优雅的方式退出锁所有权。然而，这是一种实用的方法，因此我们将其包括在这里，尽管它在这方面并不完善。

### Mutual Exclusion
最后一种预防方法是完全避免互斥。一般来说，我们知道这很困难，因为我们希望运行的代码确实有临界区。那么我们能做什么呢？

Herlihy 认为，人们可以设计各种完全不使用锁的数据结构。这些无锁（以及相关的无等待）方法背后的想法很简单：使用强大的硬件指令，您可以以不需要显式锁定的方式构建数据结构。

![[Pasted image 20241031212738.png]]

假如我们想实现原子自增，可以使用compare-and-swap 硬件指令。

![[Pasted image 20241031213121.png]]
我们不再获取锁、进行更新，然后释放锁，而是构建了一种方法，反复尝试将值更新为新值，并使用比较和交换来执行此操作。用这种方法，就不再需要lock了，也不会有死锁的发生（但是任然有livelock的可能，更加鲁棒性的解决就比这更复杂了）。

考虑稍微复杂点的例子：链表插入
![[Pasted image 20241031213516.png]]

多线程插入，会有数据竞争问题。可以通过锁来解决数据竞争：
![[Pasted image 20241031213709.png]]

使用compare-and-swap 指令将这段插入改为 lock-free 形式：
![[Pasted image 20241031214017.png]]

此处的代码更新了下一个指针，使其指向当前头部，然后尝试将新创建的节点交换到列表的新头部位置。但是，如果其他线程在此期间成功交换了新的头部，则此操作将失败，从而导致此线程使用新的头部再次尝试。

当然，构建一个有用的列表需要的不仅仅是列表插入，而且毫不奇怪，构建一个可以以无锁方式插入、删除和执行查找的列表并非易事。

## Deadlock Avoidance via Scheduling
死锁避免，不是通过破坏死锁的四个必要条件。避免死锁需要一些全局知识，了解各个线程在执行过程中可能获取哪些锁，然后以某种方式调度所述线程以确保不会发生死锁。

比如，假设我们有2个处理器和四个线程。进一步假设 线程T1会获取锁L1 和 L2（以某种顺序，在它执行的某个时刻），线程T2 会 获取L1 和 L2，见下表：

![[Pasted image 20241031214739.png]]

只要不同时运行T1和T2，就不会有死锁的发生：
![[Pasted image 20241031214816.png]]
T3 和 T1 同时运行是ok的，不会造成死锁。

![[Pasted image 20241031214921.png]]
![[Pasted image 20241031214936.png]]
这种调度策略不会造成死锁。

如您所见，静态调度导致了一种保守的方法，其中 T1、T2 和 T3 都在同一处理器上运行，因此完成作业的总时间大大延长。虽然可能可以同时运行这些任务，但对死锁的恐惧阻止我们这样做，而代价是性能。

这种方法的一个著名例子是 Dijkstra 的银行家算法，文献中描述了许多类似的方法。不幸的是，它们仅在非常有限的环境中有用，例如，在嵌入式系统中，人们完全了解必须运行的整个任务集以及它们所需的锁。此外，这种方法可能会限制并发性，正如我们在上面的第二个例子中看到的那样。因此，通过调度避免死锁并不是一个广泛使用的通用解决方案。

## Detect and Recover
最后一个通用策略是允许死锁偶尔发生，然后在检测到死锁后采取一些措施。例如，如果操作系统每年冻结一次，您只需重新启动它并愉快地（或不高兴地）继续工作。如果死锁很少见，这种非解决方案确实非常实用。许多数据库系统采用死锁检测和恢复技术。死锁检测器定期运行，构建资源图并检查是否存在循环。如果出现循环（死锁），则需要重新启动系统。如果首先需要更复杂的数据结构修复，则可能需要人工来简化该过程。

# 32.4 总结
在本章中，我们研究了并发程序中出现的错误类型。第一种类型，非死锁错误，非常常见，但通常更容易修复。它们包括原子性违规，其中应该一起执行的一系列指令没有执行，以及顺序违规，其中两个线程之间所需的顺序没有被强制执行。

我们还简要讨论了死锁：为什么会发生，以及可以做些什么。这个问题和并发本身一样古老，关于这个主题已经有数百篇论文。实践中最好的解决方案是小心谨慎，制定锁获取顺序，从而从一开始就防止死锁发生。无等待方法也有希望，因为一些无等待数据结构现在正在进入常用的库和关键系统，包括 Linux。然而，它们缺乏通用性和开发新的无等待数据结构的复杂性可能会限制这种方法的整体效用。也许最好的解决方案是开发新的并发编程模型：在 MapReduce（来自 Google）等系统中，程序员可以描述某些类型的并行计算，而无需任何锁定。锁定本质上是有问题的；也许我们应该尽量避免使用它们，除非我们真的必须这样做。

