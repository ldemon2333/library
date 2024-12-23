![[Pasted image 20241126184741.png]]


为了总结使用两级页表进行地址转换的整个过程，我们再次以算法形式呈现控制流（图 20.6）。该图显示了每次内存引用时硬件（假设是硬件管理的 TLB）中发生的情况。从图中可以看出，在发生任何复杂的多级页表访问之前，硬件首先检查 TLB；命中后，直接形成物理地址，而根本不访问页表，就像以前一样。只有在 TLB 未命中时，硬件才需要执行完整的多级查找。在这条路径上，您可以看到我们传统的两级页表的成本：两次额外的内存访问来查找有效的转换。

# 20.4 Inverted Page Tables
在页表领域中，倒置页表可以节省更多空间。在这里，我们保留一个页表，该页表包含系统每个物理页面的条目，而不是拥有多个页表（每个进程一个）。该条目告诉我们哪个进程正在使用此页面，以及该进程的哪个虚拟页面映射到此物理页面。

现在，找到正确的条目就是搜索这个数据结构。线性扫描的成本很高，因此通常在基础结构上构建哈希表以加快查找速度。PowerPC 就是这种架构的一个例子。

更一般地说，倒置页表说明了我们从一开始就说的话：页表只是数据结构。您可以使用数据结构做很多疯狂的事情，使它们变小或变大，使它们变慢或变快。多级和倒置页表只是可以做的许多事情中的两个例子。

# 20.5 Swapping the Page Tables  to Disk
Thus far, we have assumed that page tables reisde in **kernel-owned physical memory**. Thus, some systems place such page tables in **kernel virtual memory**, thereby allowing the system to swap some of these page tables to disk when memory pressure gets a little tight.

# 20.6 Summary
我们现在看到了如何构建真正的页表；不一定只是线性数组，而是更复杂的数据结构。这种表在时间和空间方面存在权衡——表越大，TLB 未命中的服务速度越快，反之亦然——因此，正确的结构选择在很大程度上取决于给定环境的约束。在内存受限的系统中（如许多旧系统），小型结构是有意义的；在具有合理内存量且工作负载积极使用大量页面的系统中，加快 TLB 未命中速度的更大表可能是正确的选择。有了软件管理的 TLB，整个数据结构空间就向操作系统创新者（提示：就是你）敞开了大门。你能想出什么新结构？它们解决了什么问题？在你入睡时想想这些问题，做只有操作系统开发人员才能做的大梦。