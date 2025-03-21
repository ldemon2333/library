从几百行 C 代码创建一个 Linux 容器的过程，一窥内核底层技术机制及真实 container runtime 的工作原理。
![[Pasted image 20250221183030.png]]
# 底层机制
如上图所示，Linux 容器由几种内核机制组成，这里将它们分到了三个维度：

1. **==`namespace`==**：一种资源视图隔离机制， 决定了进程可以看到什么，不能看到什么；例如，pid namespace 限制了进程能看到哪些其他的进程；network namespace 限制了进程能看到哪些网络设备等等。
2. **==`cgroup`==**：一种限制==资源使用量==的机制，例如一个进程最多能使用多少内存、磁盘 I/O、CPU 等等。
    - [(译) Control Group v2 (cgroupv2)（KernelDoc, 2021）](https://arthurchiao.art/blog/cgroupv2-zh/)
    - **==`setrlimit`==** 是另一种限制==资源使用量==的机制，比 `cgroups` 要老，但能做一些 cgroups 做不了的事情。
3. **==`capabilities`==**：拆分 root privilege，精细化==用户/进程权限==。
4. **==`seccomp`==**：内核的一种==安全计算== （secure computation）机制，例如限制进程只能使用某些特定的系统调用。
    
    - [wikipedia](https://en.wikipedia.org/wiki/Seccomp)
    - [man page](https://man7.org/linux/man-pages/man2/seccomp.2.html)

以上几种机制中，`cgroups` 是通过==文件系统==完成的，其他几种都是通过==系统调用==完成的。

## 1.2 Namespaces
这里只是简单列下，方面后面理解代码。其实除了 UTS，其他 namespace 都能直接从名字看出用途，

1. Mount：挂载空间，例如指定某个目录作为==进程看到的系统根目录==、独立挂载 `/proc` `/sys` 等；
2. PID：进程空间，例如最大进程数量是独立的；
3. Network (netns)：网络空间，例如能看到的网络设备是独立的，部分网络参数是独立的；
4. IPC (inter-process communication)
5. **==`UTS (UNIX Time-Sharing)`==**：让进程可以有独立的 **==host name 和 domain name==**；
6. User ID：用的不多，例如 ubuntu 22.04 / kernel 6.1 默认还是关闭的。
7. cgroup：独立的 cgroup 文件系统和资源管控；
8. Time：kernel **==`5.6+`==**。


# 代码
这里的代码来自 [Linux containers in 500 lines of code (2016)](https://blog.lizzie.io/linux-containers-in-500-loc.html)。 原文基于 Kernel 4.6，本文做了一些更新和重构。

![[Pasted image 20250221203958.png]]
1. 初始化
    1. 解析命令行参数
    2. 检查内核版本、CPU 架构等等
    3. 给容器随机生成一个 hostname
2. 创建一个 socket pair，用于容器进程和主进程之间的通信；
3. 分配栈空间，供后面 `execve()` 执行容器进程时使用；
4. 设置 cgroups 资源隔离；
5. 通过 clone 创建子进程，在里面通过 namespace/capabilities/seccomp 等技术实现资源管控和安全。

## 2.1 初始化
程序提供了三个命令行参数，

- `-u <uid>` 以什么用户权限运行；
- `-m <ctn rootfs path>` 镜像解压之后的目录；
- `-c <command>` 容器启动后运行的命令。

程序会给容器随机生成一个 hostname，就像用 `docker run` 不指定容器名字的话，docker 也会自动生成一个名字。

## 2.2 创建 socket pair，供容器进程和父进程通信
![[Pasted image 20250221204207.png]]

子进程（容器进程）需要发消息给父进程，因此初始化一个 `socketpair`。 容器进程会告诉父进程是否需要设置 uid/gid mappings， 如果需要，就会执行 `setgroups/setresgid/setresuid`。 这些是权限相关的。

## 2.3 创建 cgroup，为容器设置资源限额
cgroups 可以限制进程和进程组的资源使用量，避免有问题的容器进程影响整个系统。 它有 v1 和 v2 两个版本，
- [Documentation/cgroup-v1/cgroups.txt](https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt)
- [(译) Control Group v2 (cgroupv2 权威指南)（KernelDoc, 2021）](https://arthurchiao.art/blog/cgroupv2-zh/)

简单起见，程序里使用的 `cgroupv1`。

设置 cgroups 的逻辑比较简单，基本上就是创建 cgroup 目录，以及往 cgroups 配置文件写入配置。
上面的程序配置了以下几个资源限额：

1. `/sys/fs/cgroup/memory/$hostname/memory.limit_in_bytes=1GB`：容器进程及其子进程使用的总内存不超过 1GB；
2. `/sys/fs/cgroup/memory/$hostname/memory.kmem.limit_in_bytes=1GB`：容器进程及其子进程使用的总内存不超过 1GB；
3. `/sys/fs/cgroup/cpu/$hostname/cpu.shares=256`：CPU 总 slice 是 1024，因此限制进程最多只能占用 1/4 CPU 时间；
4. `/sys/fs/cgroup/pids/$hostname/pid.max=64`：允许容器进程及其子进程最多拥有 64 个 PID；
5. `/sys/fs/cgroup/blkio/$hostname/weight=50`：确保容器进程的 IO 优先级比系统其他进程低。
6. 降低文件描述符的 hard limit。fd 与 pid 类似，都是 per-user 的。这里设置上限之后， 后面还会 drop `CAP_SYS_RESOURCE`，因此容器内的用户是改不了的。

## 2.4 clone() 启动子进程，运行容器
![[Pasted image 20250221210443.png]]
- `flags` 规定了这个新创建的子进程要有多个独立的 namespace；
- x86 平台栈是向下增长的，因此将栈指针指向 `stack+STACK_SIZE` 作为起始位置；
- 最后还还加上了 `SIGCHLD` 标志位，这样就能 ==wait 子进程==了。

下面是子进程内做的事情：
![[Pasted image 20250221210608.png]]

### 2.4.1 `sethostname()`：感知 UTS namespace
`sethostname/gethostname` 是 Linux **==系统调用==**，用于设置或获取主机名（hostname）。

hostname 是 **==`UTS`==** namespace 隔离的， 刚才创建子进程时指定了要有独立的 UTS namespace， 因此在这个子进程内设置 hostname 时，影响的只是==这个 UTS namespace 内（即这个容器内）所有进程==看到的 hostname。

### 2.4.2 `setup_mounts()`：安全考虑，unmount 不需要的目录
![[Pasted image 20250221211039.png]]
步骤：

1. 使用 `MS_PRIVATE` 重新挂载所有内容；
2. 创建一个临时目录，并在其中创建一个子目录；
3. 将用户命令行指定的目录（`config->mount_dir`）bind mount 到临时目录上；
4. 使用 `pivot_root` 将 bind mount 作为根目录，并将旧的根目录挂载到内部临时目录上。 `pivot_root` 是一个系统调用，允许交换 `/` 处的挂载点与另一个挂载点。
5. 卸载旧的根目录，删除内部临时目录。


### 2.4.3 `setup_capabilities()`：禁用部分 capabilities
同样，基于最小权限原则，drop 所有不需要的 capabilities，


### 2.4.4 `setup_seccomp()`：禁用部分系统调用
这里将一些有安全风险的系统调用都放到了黑名单，这可能不是最好的处理方式，但能够让大家非常直观地看到底层是如何通过 seccomp 保证计算安全的。


### 2.4.5 `execve()`：执行指定的容器启动命令
前面资源视图隔离（namespace）、资源限额（cgroups）、文件目录（mount）、权限（capabilities）、安全（seccomp）等工作 都做好之后，就可以启动用户指定的容器进程了（类似于 docker 中的 entrypoint 或 command）：
![[Pasted image 20250221213344.png]]
至此，如果一切正常，容器进程就起来了。接下来我们还要容器进程与主进程的通信， 类似于真实环境中 containerd 处理来自具体容器的消息。

## 2.5 处理容器事件：user namespace 相关
Q: 容器进程与主进程如何通信的？



# 3 与真实系统对比：`ctn-d` vs. `containerd+runc`
## 3.1 容器网络
这个要花的篇幅就比较长了，常规网络方案来说分几步：

1. 在宿主机上创建一个 Linux bridge；
2. 创建一个 veth pair，一端连接到 Linux bridge，一端放到容器的 network namespace 内；
3. 配置 IP/MAC/NAT 规则等，让容器网络能连通。

可以用 `net_prio` cgroup controller 对网络资源进行限额。