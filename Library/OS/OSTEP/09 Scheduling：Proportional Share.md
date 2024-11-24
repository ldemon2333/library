Proportional-share scheduler, fair-share scheduler. Proportional-share instead of optimizing for turnaround or response time, a scheduler might instead try to guarantee that each job obtain a certain percentage of CPU time.

lottery scheduling: The basic idea is quite simple: every so often, hold a lottery to determine which process should get to run next; processes that should run more often should be given more chances to win the lottery.

# 9.8 Summary
我们介绍了比例共享调度的概念，并简要讨论了三种方法：彩票调度、步幅调度和 Linux 的完全公平调度程序 (CFS)。彩票巧妙地利用随机性来实现比例共享；步幅则确定性地实现比例共享。CFS 是本章讨论的唯一“真正的”调度程序，有点像具有动态时间片的加权循环，但可以扩展并在负载下表现良好；据我们所知，它是当今最广泛使用的公平共享调度程序。

没有一个调度程序是万能的，公平共享调度程序也有自己的问题。一个问题是，这种方法与 I/O 的配合不是特别好；如上所述，偶尔执行 I/O 的作业可能无法获得公平的 CPU 份额。另一个问题是，它们没有解决票证或优先级分配的难题，即，您如何知道应该为您的浏览器分配多少票证，或者将文本编辑器设置为什么值？其他通用调度程序（例如我们之前讨论过的 MLFQ 和其他类似的 Linux 调度程序）会自动处理这些问题，因此可能更容易部署。

好消息是，在许多领域中，这些问题并不是主要关注点，而比例共享调度程序的使用效果很好。例如，在虚拟化数据中心（或云）中，您可能希望将四分之一的 CPU 周期分配给 Windows VM，其余分配给基本 Linux 安装，比例共享可以简单而有效。