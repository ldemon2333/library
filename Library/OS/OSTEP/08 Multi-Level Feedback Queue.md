多级反馈队列（MLFQ）, the Compatible Time-Sharing System (CTSS)。解决了两个问题
- 优化了周转时间，通过首先运行最短时间，但OS并不知道一个job 会运行多久
- 其次，MLFQ 希望让系统对交互用户（即用户坐着盯着屏幕，等待进程完成）有响应能力，从而最大限度地缩短响应时间；不幸的是，像 Round Robin 这样的算法可以缩短响应时间，但对于周转时间来说却很糟糕。因此，我们的问题是：鉴于我们通常对进程一无所知，我们如何构建一个调度程序来实现这些目标？调度程序如何在系统运行时了解它正在运行的作业的特征，从而做出更好的调度决策？

提示：从历史中学习

多级反馈队列是一个从过去中学习以预测未来的系统的极好例子。这种方法在操作系统中很常见（以及计算机科学的许多其他地方，包括硬件分支预测器和缓存算法）。当作业具有行为阶段并且因此是可预测的时，这种方法有效；当然，必须小心使用此类技术，因为它们很容易出错，并导致系统做出比在完全没有知识的情况下更糟糕的决策。

# 8.1 MLFQ: Basic Rules
MLFQ 算法有几个不同的队列，每个队列分配不同的优先级，同一个队列上的job具有相同的优先级，采用时间片轮转RR算法调度。

![[Pasted image 20241104112607.png]]

MLFQ的调度算法关键是调度器如何设置优先级。Rather than giving a fixed priority to each job, MLFQ *varies* the priority of a job based on its *observed behavior*. If, for example, a job repeatedly relinquishes the CPU while waiting for input from the keyboard, MLFQ will keep its priority high, as this is how an interactive process might behave. If, instead, a job uses the CPU intensively for long periods of time, MLFQ will reduce its priority. In this way, MLFQ will try to learn about processes as they run, and thus use the history of the job to predict its future behavior. IO优先级提高，CPU优先级降低。

![[Pasted image 20241104115654.png]]

# 8.2 Attempt #1: How To Change Priority
a mix of interactive jobs that are short-running (and may frequently relinquish the CPU), and some longer-running “CPU-bound” jobs that need a lot of CPU time but where response time isn’t important.

作业的allotment，the allotment is the amount of time a job can spend at a given priority level before the scheduler reduces its priority. 为了简化，首先，我们假设the allotment 等于一个时间片。

Here is our first attempt at a priority-adjustment algorithm:
- Rule 3: When a job enters the system, it is placed at the highest priority (the topmost queue).
- Rule 4a: 如果一个job 用完了它的allotment，它的优先级就会减少。
- Rule 4b: 如果job 放弃了CPU（比如，执行IO操作）在它的allotment 用完之前，它的优先级保留。

问题：饥饿问题，如果有大量的交互型jobs，他们结合起来消耗所有CPU时间，导致长时间jobs会饥饿。

欺骗调度程序通常是指通过一些狡猾的手段，欺骗调度程序，使其给予您超过应得份额的资源。我们描述的算法容易受到以下攻击：在使用分配之前，发出 I/O 操作（例如，对文件），从而放弃 CPU；这样做可以让您保持在同一队列中，从而获得更高百分比的 CPU 时间。如果做得正确（例如，在放弃 CPU 之前运行 99% 的分配），作业几乎可以垄断 CPU。

最后，程序可能会随着时间的推移改变其行为；原本受 CPU 限制的作业可能会转变为交互阶段。按照我们目前的方法，这样的作业将不会像系统中的其他交互式作业一样被处理。

# 8.3 Attempt #2: The Priority Boost
提升jobs的优先级，一开始将所有的jobs丢入最高优先级队列中；
- Rule 5：经过一段时间 S 后，将所有 jobs 丢入最高优先级队列中
新规则解决了两个问题。
- 进程保证不会被饥饿
- If a CPU-bound job has become interactive, the schedular treats it properly once it has received the priority boos.

添加时间段 S 会引出一个显而易见的问题：应该将 S 设置为多少？著名系统研究员 John Ousterhout 过去常常将系统中的此类值称为巫术常量，因为它们似乎需要某种形式的黑魔法才能正确设置它们。不幸的是，S 就是这样。如果设置得太高，长时间运行的作业可能会饿死；如果设置得太低，交互式作业可能无法获得适当的 CPU 份额。因此，通常由系统管理员来找到正确的值 - 或者在现代世界中，越来越多地使用基于机器学习的自动方法。

# 8.4 Attempt #3: Better Accounting
现在我们还有一个问题需要解决：如何防止调度程序被玩弄？您可能已经猜到了，真正的罪魁祸首是规则 4a 和 4b，它们允许作业在分配到期之前放弃 CPU，从而保留其优先级。那么我们应该怎么做呢？

这里的解决方案是更好地核算 MLFQ 各级的 CPU 时间。调度程序不应忘记进程在执行 I/O 时在给定级别使用了多少分配量，而应跟踪该情况；一旦进程使用了​​其分配量，就会将其降级到下一个优先级队列。它是在一次长突发还是多次短突发中使用其分配量并不重要。因此，我们将规则 4a 和 4b 重写为以下单个规则：

- Rule 4: 一旦一个job 使用完了它的时间allotment在一个优先级上（无论它放弃了多少次CPU），它的优先级就会降低。

# 8.5 Tuning MLFQ And Other Issues
参数的调优与设置，有多少个队列，每个队列的时间片应该设置多大，分配量多大？

# 8.6 总结
![[Pasted image 20241104124106.png]]
