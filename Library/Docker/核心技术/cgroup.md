# cgroup 是什么
cgroup 是一种==以 hierarchical（树形层级）方式组织进程的机制==（a mechanism to organize processes hierarchically），以及在层级中==以受控和 可配置的方式==（controlled and configurable manner）**==分发系统资源==** （distribute system resources）。

cgroups，其名称源自控制组群（control groups）的缩写，是内核的一个特性，用于限制、记录和隔离一组进程的资源使用（CPU、内存、磁盘 I/O、网络等）

**资源限制**：可以配置 cgroup，从而限制进程可以对特定资源（例如内存或 CPU）的使用量

**优先级** ：当资源发生冲突时，您可以控制一个进程相比另一个 cgroup 中的进程可以使用的资源量（CPU、磁盘或网络）

**记录**：在 cgroup 级别监控和报告资源限制

**控制**：您可以使用单个命令更改 cgroup 中所有进程的状态（冻结、停止或重新启动）

Cgroups功能的实现依赖于四个核心概念：**子系统、控制组、层级树、任务**

**控制组（cgroup）**  
表示一组进程和一组带有参数的子系统的关联关系。例如，一个进程使用了 CPU 子系统来限制 CPU 的使用时间，则这个进程和 CPU 子系统的关联关系称为控制组

**层级树（hierarchy）**  
由一系列的控制组按照树状结构排列组成的。这种排列方式可以使得控制组拥有父子关系，子控制组默认拥有父控制组的属性，也就是子控制组会继承于父控制组。比如，系统中定义了一个控制组 c1，限制了 CPU 可以使用 1 核，然后另外一个控制组 c2 想实现既限制 CPU 使用 1 核，同时限制内存使用 2G，那么 c2 就可以直接继承 c1，无须重复定义 CPU 限制

**子系统（subsystem）**

一个内核的组件，一个子系统代表一类资源调度控制器。例如内存子系统可以限制内存的使用量，CPU 子系统可以限制 CPU 的使用时间。子系统是真正实现某类资源的限制的基础

Subsystem(子系统) cgroups 中的子系统就是一个资源调度控制器(又叫 controllers)

在/sys/fs/cgroup/这个目录下可以看到cgroup子系统

- **cpu**：使用调度程序控制任务对cpu的使用
- **cpuacct**：自动生成cgroup中任务对cpu资源使用情况的报告
- **cpuset**：可以为cgroup中的任务分配独立的cpu和内存
- **blkio**：可以为块设备设定输入 输出限制，比如物理驱动设备
- **devices**：可以开启或关闭cgroup中任务对设备的访问
- **freezer**： 可以挂起或恢复cgroup中的任务
- **pids**：限制任务数量
- **memory**：可以设定cgroup中任务对内存使用量的限定，并且自动生成这些任务对内存资源使用情况的报告
- **perf_event**：使用后使cgroup中的任务可以进行统一的性能测试
- **net_cls**：docker没有直接使用它，它通过使用等级识别符标记网络数据包，从而允许linux流量控制程序识别从具体cgroup中生成的数据包

**任务（task）** 

在cgroup中，任务就是一个进程，一个任务可以是多个cgroup的成员，但这些cgroup必须位于不同的层级，子进程自动成为父进程cgroup的成员，可按需求将子进程移到不同的cgroup中
![[Pasted image 20250222130751.png]]

cgroup 的作用基本上就是控制一个进程或一组进程可以访问或使用给定关键资源（CPU、内存、网络和磁盘 I/O）的量。一个容器中通常运行了多个进程，并且您需要对这些进程实施统一控制，因此 cgroup 是容器的关键组件。Kubernetes 环境使用cgroup 在 pod 级别上部署[资源请求和限制](https://link.zhihu.com/?target=https%3A//github.com/kubernetes/community/blob/master/contributors/design-proposals/node/node-allocatable.md%23recommended-cgroups-setup)以及对应的 QoS 类

下图说明了当您将特定比例的可用系统资源分配给一个 cgroup（在本例中，为cgroup‑1）后，剩余资源是如何在系统上其他 cgroup（以及各个进程）之间进行分配的
![[Pasted image 20250222131006.png]]


## cgroup 组成部分
cgroup 主要由两部分组成：
- 核心（core）：主要负责层级化地组织进程；
- 控制器（controllers）：

cgroup 到目前为止，有两个大版本， cgroup v1 和 v2 。以下内容以 cgroup v2 版本为主，涉及两个版本差别的地方会在下文详细介绍。

cgroup 主要限制的资源是：

- CPU
    
- 内存
    
- 网络
    
- 磁盘 I/O

cgroup 代表“控制组”，并且不会使用大写。cgroup 是一种分层组织进程的机制， 沿层次结构以受控的方式分配系统资源。我们通常使用单数形式用于指定整个特征，也用作限定符如 “cgroup controller” 。

cgroup 主要有两个组成部分：

- core - 负责分层组织过程；
- controller - 通常负责沿层次结构分配特定类型的系统资源。每个 cgroup 都有一个 `cgroup.controllers` 文件，其中列出了所有可供 cgroup 启用的控制器。当在 `cgroup.subtree_control` 中指定多个控制器时，要么全部成功，要么全部失败。在同一个控制器上指定多项操作，那么只有最后一个生效。每个 cgroup 的控制器销毁是异步的，在引用时同样也有着延迟引用的问题；

所有 cgroup 核心接口文件都以 `cgroup` 为前缀。每个控制器的接口文件都以控制器名称和一个点为前缀。控制器的名称由小写字母和“_ ” 组成，但永远不会以“ _ ” 开头。

## cgroup 的核心文件
- cgroup.type - （单值）存在于非根 cgroup 上的可读写文件。通过将“threaded”写入该文件，可以将 cgroup 转换为线程 cgroup，可选择 4 种取值，如下：
    
    - 1. domain - 一个正常的有效域 cgroup
    - 2. domain threaded - 线程子树根的线程域 cgroup
    - 3. domain invalid - 无效的 cgroup
    - 4. threaded - 线程 cgroup，线程子树
- cgroup.procs - （换行分隔）所有 cgroup 都有的可读写文件。每行列出属于 cgroup 的进程的 PID。PID 不是有序的，如果进程移动到另一个 cgroup ，相同的 PID 可能会出现不止一次；
    
- cgroup.controllers - （空格分隔）所有 cgroup 都有的只读文件。显示 cgroup 可用的所有控制器；
    
- cgroup.subtree_control - （空格分隔）所有 cgroup 都有的可读写文件，初始为空。如果一个控制器在列表中出现不止一次，最后一个有效。当指定多个启用和禁用操作时，要么全部成功，要么全部失败。
    
    - 1. 以“+”为前缀的控制器名称表示启用控制器
    - 2. 以“-”为前缀的控制器名称表示禁用控制器
- cgroup.events - 存在于非根 cgroup 上的只读文件。
    
    - 1. populated - cgroup 及其子节点中包含活动进程，值为1；无活动进程，值为0.
    - 2. frozen - cgroup 是否被冻结，冻结值为1；未冻结值为0.
- cgroup.threads - （换行分隔）所有 cgroup 都有的可读写文件。每行列出属于 cgroup 的线程的 TID。TID 不是有序的，如果线程移动到另一个 cgroup ，相同的 TID 可能会出现不止一次。
    
- cgroup.max.descendants - （单值）可读写文件。最大允许的 cgroup 数量子节点数量。
    
- cgroup.max.depth - （单值）可读写文件。低于当前节点最大允许的树深度。
    
- cgroup.stat - 只读文件。
    
    - 1. nr_descendants - 可见后代的 cgroup 数量。
    - 2. nr_dying_descendants - 被用户删除即将被系统销毁的 cgroup 数量。
- cgroup.freeze - （单值）存在于非根 cgroup 上的可读写文件。默认值为0。当值为1时，会冻结 cgroup 及其所有子节点 cgroup，会将相关的进程关停并且不再运行。冻结 cgroup 需要一定的时间，当动作完成后， cgroup.events 控制文件中的 “frozen” 值会更新为“1”，并发出相应的通知。cgroup 的冻结状态不会影响任何 cgroup 树操作（删除、创建等）；
    
- cgroup.kill - （单值）存在于非根 cgroup 上的可读写文件。唯一允许值为1，当值为1时，会将 cgroup 及其所有子节点中的 cgroup 杀死（进程会被 SIGKILL 杀掉）。一般用于将一个 cgroup 树杀掉，防止叶子节点迁移；

## cgroup 的归属和迁移
系统中的每个进程都属于一个 cgroup，一个进程的所有线程都属于同一个 cgroup。一个进程可以从一个 cgroup 迁移到另一个 cgroup 。进程的迁移不会影响现有的后代进程所属的 cgroup。
![[Pasted image 20250222125615.png]]
**图 5 ，进程及其子进程的 cgroup 分配；跨 cgroup 迁移示例**

跨 cgroup 迁移进程是一项代价昂贵的操作并且有状态的资源限制（例如，内存）不会动态的应用于迁移。因此，经常跨 cgroup 迁移进程只是作为一种手段。不鼓励直接应用不同的资源限制。

## 如何实现跨 cgroup 迁移
每个cgroup都有一个可读写的接口文件 “cgroup.procs” 。每行一个 PID 记录 cgroup 限制管理的所有进程。一个进程可以通过将其 PID 写入另一 cgroup 的 “cgroup.procs” 文件来实现迁移。

但是这种方式，只能迁移一个进程在单个 write(2) 上的调用（如果一个进程有多个线程，则会同时迁移所有线程，但也要参考线程子树，是否有将进程的线程放入不同的 cgroup 的记录）。

当一个进程 fork 出一个子进程时，该进程就诞生在其父亲进程所属的 cgroup 中。

一个没有任何子进程或活动进程的 cgroup 是可以通过删除目录进行销毁的（即使存在关联的僵尸进程，也被认为是可以被删除的）。

# 什么是 cgroups
当明确提到多个单独的控制组时，才使用复数形式 “cgroups” 。

cgroups 形成了树状结构。（一个给定的 cgroup 可能有多个子 cgroup 形成一棵树结构体）每个非根 cgroup 都有一个 `cgroup.events` 文件，其中包含 `populated` 字段指示 cgroup 的子层次结构是否具有实时进程。所有非根的 `cgroup.subtree_control` 文件，只能包含在父级中启用的控制器。
![[Pasted image 20250222125911.png]]
**图 6 ，cgroups 示例**

如图所示，cgroup1 中限制了使用 cpu 及 内存资源，它将控制子节点的 CPU 周期和内存分配（即，限制 cgroup2、cgroup3、cgroup4 中的cpu及内存资源分配）。cgroup2 中启用了内存限制，但是没有启用cpu的资源限制，这就导致了 cgroup3 和 cgroup4 的内存资源受 cgroup2中的 mem 设置内容的限制；cgroup3 和 cgroup4 会自由竞争在 cgroup1 的 cpu 资源限制范围内的 cpu 资源。

由此，也可以明显的看出 cgroup 资源是自上而下分布约束的。只有当资源已经从上游 cgroup 节点分发给下游时，下游的 cgroup 才能进一步分发约束资源。所有非根的 `cgroup.subtree_control` 文件只能包含在父节点的 `cgroup.subtree_control` 文件中启用的控制器内容

那么，**子节点 cgroup 与父节点 cgroup 是否会存在内部进程竞争的情况呢**？

当然不会。cgroup v2 中，设定了非根 cgroup 只能在没有任何进程时才能将域资源分发给子节点的 cgroup。简而言之，只有不包含任何进程的 cgroup 才能在其 `cgroup.subtree_control` 文件中启用域控制器，这就保证了，进程总在叶子节点上。


# 子系统接口/参数
### 1.1.1、cpu子系统：于限制进程的 CPU 利用率

**cpu.shares**：cpu比重分配。通过一个整数的数值来调节cgroup所占用的cpu时间。例如，有2个cgroup（假设为CPU1，CPU2），其中一个(CPU1)cpu.shares设定为100另外一个(CPU2)设为200，那么CPU2所使用的cpu时间将是CPU1所使用时间的2倍。cpu.shares 的值必须为2或者高于2

**cpu.cfs_period_us**：规定CPU的时间周期(单位是微秒)。最大值是1秒，最小值是1000微秒。如果在一个单CPU的系统内，要保证一个cgroup 内的任务在1秒的CPU周期内占用0.2秒的CPU时间，可以通过设置cpu.cfs_quota_us 为200000和cpu.cfs_period_us 为 1000000

**cpu.cfs_quota_us**：在单位时间内（即cpu.cfs_period_us设定值）可用的CPU最大时间（单位是微秒）。cpu.cfs_quota_us值可以大于cpu.cfs_period_us值，例如在一个双CPU的系统内，想要一个cgroup内的进程充分的利用2个CPU，可以设定cpu.cfs_quota_us为 200000 及cpu.cfs_period_us为 100000



