Proportional-share scheduler, fair-share scheduler. Proportional-share instead of optimizing for turnaround or response time, a scheduler might instead try to guarantee that each job obtain a certain percentage of CPU time.

lottery scheduling: The basic idea is quite simple: every so often, hold a lottery to determine which process should get to run next; processes that should run more often should be given more chances to win the lottery.

## Crux: How to share the CPU proportionally
How can we design a scheduler to share the CPU in a proportional manner? What are the key mechanisms for doing so? How effective are they?

# 9.1 Basic Concept: Tickets Represent Your Share
Tickets, which are used to represent the share of a resource that a process should receive.

An example. Two processes, A and B, and further that A has 75 tickets while B has only 25. Thus, what we would like is for A to receive 75% of the CPU and B the remaining 25%.

![[Pasted image 20241125144823.png]]

# 9.7 The Linux Completely Fair Scheduler(CFS)
The **Completely Fair Scheduler** (of CFS), implements fair-share scheduling, but does so in a highly efficient and scalable manner.

To achieve efficiency goals, CFS aims to spend very little time making scheduling decisions, through both its inherent design and its clever use of data structures well-suited to the task.

## Basic Operation
Its goal is simple: to fairly divide a CPU evenly among all competing porcesses. It does so through a simple counting-based technique known as **virtual runtime** (vruntime).

As each process runs, it accumulates vruntime. In the most basic case, each process's vruntime increases at the same rate, in proportion with physical (real) time. When a scheduling decision occurs, CFS will pick the process with the lowest vruntime to run next.

This raises a question: how does the scheduler know when to stop the currently running process, and run the next one? The tension here is clear: if CFS switches too often, fairness is increased, as CFS will ensure that each process receives its share of CPU even over miniscule time windows, but at the cost of performance (too much context switching); if CFS switches less often, performance is increased (reduced context switching), but at the cost of near-term fairness.

CFS manages this tension through various control parameters. The first is **sched_latency**. CFS uses this value to determine how long one process should run before considering a switch (effectively determinig its time slice but in a dynamic fashion). A typical **sched_latency** value is 48 (millisconds); CFS divides this value by the number (n) of processes running on the CPU to determine the time slice for a process, and thus ensures that over this period of time, CFS will be completely fair.

For example, if there are n = 4 processes running, CFS divides the value of sched_latency by n to arrive at a per-process time slice of 12 ms. CFS then schedules the first job and runs it until it has used 12 ms of (virtual) runtime, and then checks to see if there is a job with lower vruntime to run instead. In this case, there is, and CFS would switch to one of the three other jobs, and so forth. Figure 9.4 shows an example where the four jobs (A, B, C, D) each run for two time slices in this fashion; two of them (C, D) then complete, leaving just two remaining, which then each run for 24 ms in round-robin fashion.

![[Pasted image 20241125151534.png]]
But what if there are "too many" process running? Wouldn't that lead to too small of a time slice, and thus too many context switches? And the answer is yes.

To address this issue, CFS adds another parameter, **min_granularity**, which is usually set to a vlaue like 6ms. CFS wil never set the time slice of a process to less than this value, ensuring that not too much time is spent in scheduling overhead.

For example, if there are ten processes running, out original calculation would divide sched_latency by ten to determine the time slice (4.8ms). However, because of min_granularity, CFS will set the time slice of each porcess to 6ms instead. Although CFS won’t (quite) be perfectly fair over the target scheduling latency (sched latency) of 48 ms, it will be close, while still achieving high CPU efficiency.

Note that CFS utilizes a periodic timer interrupt, which means it can only make decisions at fixed time intervals. This interrupt goes off frequently (e.g., every 1ms), giving CFS a chance to wake up and determine if the current job has reached the end of its run. If a job has a time slice that is not a perfect multiple of the timer interrupt interval, that is OK; CFS tracks vruntime precisely, which means that over the long haul, it will eventually approximate ideal sharing of the CPU

## Weighting (Niceness)
CFS also enables controls over process priority, enabling users or administrators to give some processes a higher share of the CPU. It does this not with tickets, but through a classic UNIX mechanism known as the **nice** level of a process. The nice parameter can be set anywhere from -20 to +19 for a process, with a default of 0. Positive nice values imply *lower* priority and negative values imply *higher* priority; when you’re too nice, you just don’t get as much (scheduling) attention, alas.

CFS maps the nice value of each process to a weight, as shown here:
![[Pasted image 20241125152441.png]]

These weights allow us to compute the effective time slice of each process (as we did before), but now accounting for their priority differences. The formula used to do so is as follows, assuming n process:
![[Pasted image 20241125152555.png]]
An example. Assume there are two jobs, A and B. A, because it's our most precious job, is given a higher priority by assigning it nice value of -5; B, just has the default priority (nice value equal to 0). This means weightA is 3121, whereas weightB is 1024. 

In addition to generalizing the time slice calculation, the way CFS calculates vruntime must also be adapted. Here is the new formula, which takes the actual run time that process i has accrued (runtimei) and scales it inversely by the weight of the process, by dividing the default weight of 1024 (weight0) by its weight, weighti. In our running example, A's vruntime will accumulate at one-third the rate of B's.

![[Pasted image 20241125153144.png]]

## Using Red-Black Trees
CFS does not keep all processes in this structure; rather, only running (or runnable) processes are kept therein. If a process goes to sleep (say, waiting on an I/O to complete, or a network packet to arrive), it is removed from the tree and kept track of elsewhere.

Let’s look at an example to make this more clear. Assume there are ten jobs, and that they have the following values of vruntime: 1, 5, 9, 10, 14, 18, 17, 21, 22, and 24. If we kept these jobs in an ordered list, finding the next job to run would be simple: just remove the first element. However, when placing that job back into the list (in order), we would have to scan the list, looking for the right spot to insert it, an O(n) operation.

Keeping the same values in a red-black tree makes most operations more efficient, as depicted in Figure 9.5. Processes are ordered in the tree by vruntime, and most operations (such as insertion and deletion) are logarithmic in time, i.e., O(log n). When n is in the thousands, logarithmic is noticeably more efficient than linear.

![[Pasted image 20241125154434.png]]


## Dealing With I/O and sleeping processes
CFS 解决的这个问题是在进程长时间休眠后唤醒时可能会导致“饿死”的情况。例如，假设有两个进程 A 和 B，其中 A 持续运行，而 B 长时间休眠（比如 10 秒钟）。当 B 醒来时，它的 `vruntime` 会比 A 慢 10 秒钟，因此如果不加以处理，B 可能会在接下来的 10 秒内独占 CPU，导致 A 被饿死。

### CFS 如何解决这个问题：

CFS 通过在进程唤醒时调整它的 `vruntime` 来避免这种情况。具体来说，当一个进程（如 B）从休眠中醒来时，CFS 会将该进程的 `vruntime` 设置为红黑树中最小的 `vruntime`，即当前正在运行的进程的 `vruntime`。红黑树中的进程都在运行，因此这种方式确保了唤醒的进程不会因为 `vruntime` 太大而立刻占用 CPU 时间。

#### 关键效果：

1. **避免饿死**：通过将唤醒进程的 `vruntime` 设置为最小值，CFS 确保了进程 B 唤醒后不会立即独占 CPU。它将像其他进程一样公平地参与调度，避免了饿死。
    
2. **公平性**：这种调整防止了进程在休眠期间积累过多的 `vruntime`，确保每个进程在公平的条件下分配 CPU 时间。
    

#### 成本：

1. **短时间休眠进程可能得不到公平的 CPU 时间**：由于 CFS 在进程唤醒时将 `vruntime` 设置为最小值，休眠时间较短的进程可能永远无法完全公平地得到 CPU 时间，因为它们的 `vruntime` 经常被归零，从而导致它们在长时间休眠后反复被抢占。




CFS（完全公平调度器，Completely Fair Scheduler）是 Linux 内核中的调度器，用于在多核处理器系统中分配 CPU 时间给进程，确保进程得到公平的 CPU 使用时间。

### CFS 工作原理的关键概念：

1. **虚拟运行时间（vruntime）**： CFS 的核心思想是确保每个进程的运行时间相对公平。每个进程都有一个 `vruntime` 值，表示该进程在调度器中的虚拟运行时间。进程的 `vruntime` 会随着它在 CPU 上运行而增加，增加的速度与进程的优先级有关。低优先级的进程 `vruntime` 增加得更快，高优先级的进程 `vruntime` 增加得更慢。
    
2. **红黑树**： CFS 使用红黑树（Red-Black Tree）来管理可运行的进程队列。树的每个节点代表一个进程，节点的排序基于进程的 `vruntime` 值。根节点代表 `vruntime` 最小的进程，也就是下一个会被调度执行的进程。这样，CFS 保证每次调度时会选择 `vruntime` 最小的进程，这个进程是最“公平”的。
    
3. **时间片（Time slice）**： 与传统的轮转调度算法不同，CFS 不采用固定的时间片。它根据每个进程的优先级（即 `vruntime`）动态决定调度的时间。进程的优先级越高，`vruntime` 增加得越慢，因此它会在 CPU 上运行得更久，反之，低优先级进程 `vruntime` 增加得较快，它的调度间隔相对较短。
    
4. **公平性与调度延迟**： CFS 的设计目标是使得所有进程在理论上得到相同的 CPU 时间。为了避免某些进程长时间无法获得 CPU 时间，CFS 会在调度时加入一定的“延迟”，确保没有进程被饥饿（即永远得不到 CPU 时间）。进程的运行时间会通过 `vruntime` 来追踪，调度器根据 `vruntime` 值来做出调度决策。
    
5. **负载均衡**： 在多核系统中，CFS 还需要进行负载均衡，确保 CPU 的使用效率。在负载均衡过程中，CFS 会根据各个 CPU 核心上进程的负载情况，决定是否需要将进程从一个核心迁移到另一个核心。CFS 会尽量确保每个 CPU 核心上的负载尽可能均衡。
    

### 总结：

CFS 的主要特点是使用 `vruntime` 来追踪进程的“虚拟”运行时间，基于红黑树的数据结构选择下一个需要调度的进程，并且保证每个进程能公平地得到 CPU 时间。通过动态调整时间片和优先级，CFS 能够有效平衡进程的调度和系统的负载均衡。

# 9.8 Summary
我们介绍了比例共享调度的概念，并简要讨论了三种方法：彩票调度、步幅调度和 Linux 的完全公平调度程序 (CFS)。彩票巧妙地利用随机性来实现比例共享；步幅则确定性地实现比例共享。CFS 是本章讨论的唯一“真正的”调度程序，有点像具有动态时间片的加权循环，但可以扩展并在负载下表现良好；据我们所知，它是当今最广泛使用的公平共享调度程序。

没有一个调度程序是万能的，公平共享调度程序也有自己的问题。一个问题是，这种方法与 I/O 的配合不是特别好；如上所述，偶尔执行 I/O 的作业可能无法获得公平的 CPU 份额。另一个问题是，它们没有解决票证或优先级分配的难题，即，您如何知道应该为您的浏览器分配多少票证，或者将文本编辑器设置为什么值？其他通用调度程序（例如我们之前讨论过的 MLFQ 和其他类似的 Linux 调度程序）会自动处理这些问题，因此可能更容易部署。

好消息是，在许多领域中，这些问题并不是主要关注点，而比例共享调度程序的使用效果很好。例如，在虚拟化数据中心（或云）中，您可能希望将四分之一的 CPU 周期分配给 Windows VM，其余分配给基本 Linux 安装，比例共享可以简单而有效。