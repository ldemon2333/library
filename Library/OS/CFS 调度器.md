
## 1.2 CFS 扩展
最初的 CFS 管理的是单个任务（进程）的调度，给每个进程分配公平的 CPU 时间。 但很多情况下，进程会组织成进程组（task group）的形式， 用户希望先对进程组分配 CPU 份额，再**==在每个进程组里面实现公平调度==**。

举个具体例子，多个用户使用同一台机器时，可能希望，

- 首先按 user 公平（也可以不公平）分配 CPU；
- 针对每个 user，再对其所有进程公平分配这个 user 的总 CPU 时间。

为此，**==CFS 引入了几项扩展==**，例如

- 实时任务的组调度（RT group）
- 常规进程的组调度（task group）

但实现这几个扩展是需要一些前提的。

### 1.2.1 前提：`CONFIG_CGROUPS`
要实现按进程组分配和管理 CPU 份额的功能，首先要能够控制（**==control==**） 进程组（task **==group==**）的资源限额， 这种技术在 Linux 内核中已经有了，叫==控制组（control group）==，缩写是 cgroup。 （**==`CONFIG_CGROUPS`==**）。

- cgroup 有两个版本，分别称为 cgroup v1 和 cgroup v2。这两个版本**==不兼容==**，现在默认都是用的 v1；
- 有了 `cgroup`，调度器就能通过 `cgroup` 伪文件系统来管理进程组占用的资源（我们这里关心的是CPU 资源）了；
- 更多信息见 Documentation/admin-guide/cgroup-v1/cgroups.rst。

### 1.2.2 前提：`CONFIG_CGROUP_SCHED`
cgroup 是按资源类型（cpu/memory/device/hugetlb/…）来做资源限额的，每种资源 类型会有一种对应的控制器（controller），有独立的开关。 控制**==进程或进程组能使用的 CPU 时间==**，对应的开关是 `CONFIG_CGROUP_SCHED`。

至此，支持进程组级别资源控制的基础就具备了。接下来就是 CFS 扩展代码的实现， 添加对于 realtime/conventional task group 的支持。下面分别来看下。

### 1.2.3 扩展：支持实时进程组（`CONFIG_RT_GROUP_SCHED`）

`CONFIG_RT_GROUP_SCHED` 支持对 real-time (SCHED_FIFO and SCHED_RR) 任务进行分组 CFS。

实时进程有严格的响应时间限制，不管机器的 load 有多高，都应该确保这些进程的响应实时性。 例子：内核中的 **==`migration`==** 进程，负责在不同 CPU 之间分发任务（进程负载均衡）。

```
$ ps -ef | grep migration
root          12       2  0       00:00:01 [migration/0]
```

### 1.2.4 扩展：支持常规进程组（`CONFIG_FAIR_GROUP_SCHED`）
实时进程之外的进程就是所谓的==常规进程==，它们没有严格的响应时间限制， 当系统繁忙时，响应延迟就会增加。

在 cgroup 技术基础上上，再对 CFS 代码做一些增强，就能够支持进程组内的公平调度了。 这些增强代码是通过编译选项 **==`CONFIG_FAIR_GROUP_SCHED`==** 控制的。 支持对普通 CFS (SCHED_NORMAL, SCHED_BATCH) 任务进行分组。

至此，我们已经能对进程和进程组进行 CFS 调度。

# 2 CFS 相关设计
