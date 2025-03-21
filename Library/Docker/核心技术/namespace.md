# What are namespaces?
Namespaces are a Linux kernel feature released in kernel version 2.6.24 in 2008. They provide processes with **their own system view**, thus **isolating independent processes from each other**. In other words, **namespaces define the set of resources that a process can use**.

A powerful side of namespaces is that they limit access to system resources without the running process being aware of the limitations. In typical Linux fashion they are represented as files under the `/proc/<pid>/ns` directory.

When we spawn a new process all the namespaces are inherited from its parent.

Namespaces are created with the _clone_ syscall with one of the following arguments:
- CLONE_NEWNS - create new mount namespace;
    
- CLONE_NEWUTS - create new UTS namespace;
    
- CLONE_NEWIPC - create new IPC namespace;
    
- CLONE_NEWPID - create new PID namespace;
    
- CLONE_NEWNET - create new NET namespace;
    
- CLONE_NEWUSER - create new USR namespace;
    
- CLONE_NEWCGROUP - create a new cgroup namespace.

Namespaces can also be created using the _unshare_ syscall. The difference between _clone_ and _unshare_ is that _clone_ spawns a new process inside a new set of namespaces, and _unshare_ moves the current process inside a new set of namespaces (unshares the current ones).

# Why use namespaces?
Moreover, namespaces can provide even fine-grained isolation, allowing process A and B to share some system resources (e.g. sharing a mount point or a network stack). Namespaces are often used when untrusted code has to be executed on a given machine without compromising the host OS. The difference between containers and VMs is that containers share and use directly the host OS kernel, thus making them significantly lighter than virtual machines as there is no hardware emulation.


# Types of namespaces
In the current stable Linux Kernel version 5.7 there are seven different namespaces:

- **PID namespace**: isolation of the system process tree;
    
- **NET namespace**: isolation of the host network stack;
    
- **MNT namespace**: isolation of host filesystem mount points;
    
- **UTS namespace**: isolation of hostname;
    
- **IPC namespace**: isolation for interprocess communication utilities (shared segments, semaphores);
    
- **USER namespace**: isolation of system users IDs;
    
- **CGROUP namespace**: isolation of the virtual cgroup filesystem of the host.

The namespaces are per-process attributes. **Each process can perceive at most one namespace**. In other words, at any given moment, any process P belongs to exactly one instance of each namespace. 

## PID
Historically, the Linux kernel has maintained a single process tree. The tree data structure contains a reference to every process currently running in a parent-child hierarchy. It also enumerates all running processes in the OS. This structure is maintained in the so called _procfs_ filesystem which is a property of the live system (i.e. it’s present only when the OS is running).

On system boot, the first process started on most of the modern Linux OS is systemd (system daemon), which is situated on the root node of the tree. Its parent is **PID=0 which is a non-existing process in the OS**. This process is after that responsible for starting the other services/daemons, which are represented as its childs and are necessary for the normal functioning of the OS. These processes will have PIDs > 1 and the PIDs in the tree structure are unique.

With the introduction of the _Process namespace (or PID namespace)_ it became possible to make nested process trees. It allows processes other than systemd (PID=1) to perceive themselves as the root process by moving on the top of a subtree, thus obtaining PID=1 in that subtree. All processes in the same subtree will also obtain IDs relative to the process namespace. This also means that some processes may end up having multiple IDs depending on the number of process namespaces that they are in. Yet, in each namespace, at most one process can have a given PID.
![[Pasted image 20250219183541.png]]
In the Linux kernel the PID is represented as a structure. Inside we can also find the namespaces a process is part of as an array of _upid struct_.
![[Pasted image 20250219184744.png]]
To create a new process inside a new PID namespace, one must call the _clone()_ system call with a special flag **CLONE_NEWPID**. Whereas the other namespaces discussed below can also be created using the _unshare()_ system call, a PID namespace can only be created at the time a new process is spawned using _clone()_ or _fork()_ syscalls.
![[Pasted image 20250219184046.png]]
What happened? It seems like the shell is stuck between the two namespaces. This is due to the fact that unshare doesn’t enter the new namespace after being executed (execve() call). This is the desired Linux kernel behavior. The current “unshare” process calls the unshare system call, creating a new pid namespace, but the current “unshare” process is not in the new pid namespace. A process B creates a new namespace but the process B itself won’t be put into the new namespace, only the sub-processes of process B will be put into the new namespace. After the creation of the namespace the unshare program will execute /bin/bash. Then /bin/bash will fork several new sub-processes to do some jobs. These sub-processes will have a PIDs relative to the new namespace and when these processes are done they will exit leaving the namespace without PID=1. The Linux kernel doesn’t like to have PID namespaces without a process with PID=1 inside. So when the namespace is left empty the kernel will disable some mechanisms which are related to the PID allocation inside this namespace thus leading to this error. This error is well documented if you [look](https://stackoverflow.com/questions/44666700/unshare-pid-bin-bash-fork-cannot-allocate-memory) around the Internet.

Instead, we must instruct the unshare program to fork a new process after it has created the namespace. Then this new process will have PID=1 and will execute our shell program. In that way when the sub-processes of /bin/bash exit the namespace will still have a process with PID=1.
![[Pasted image 20250219192238.png]]
But why doesn't our shell have PID 1 when we use ps? And why do we still see the process from the root namespace ? The ps program uses the _procfs_ virtual file system to obtain information about the current processes in the system. This filesystem is mounted in the _/proc_ directory. However, in the new namespace this mountpoint describes the processes from the root PID namespace. There are two ways to avoid that:
![[Pasted image 20250219192602.png]]

**To sum up about the process namespace**:
- Processes within a namespace only see (interact with) the processes in the same PID namespace (isolation);
- Each PID namespace has its own numbering starting at 1 (relative);
- This numbering is unique per process namespace - If PID 1 goes away then the whole namespace is deleted;
- Namespaces can be nested;
- A process ends up having multiple PIDs (when namespaces are nested);
- All ‘ps’-like commands use the virtual procfs file system mount to deliver their functionalities.

## NET 
A network namespace limits the view of a process of the host network. It allows a process to have its own separation from the host network stack (set of network interfaces, routing rules, set of netfilter hooks). Let’s inspect that:
![[Pasted image 20250219194319.png]]
Let’s now create a fresh new network namespace and inspect the network stack.
![[Pasted image 20250219194331.png]]
We can see that the entire network stack of the process has changed. There is only the loopback interface which is also down. Said with other words, this process is unreachable via the network. But that’s a problem, isn’t it? Why do we need a virtually isolated network stack if we can’t communicate through it? Here is an illustration of the situation:
![[Pasted image 20250219194359.png]]
As normally we want to be able to communicate in some way with a given process, we have to provide a way to connect different net namespaces.

### Connecting a pair of namespaces
In order to make a process inside a new network namespace reachable from another network namespace, a pair of virtual interfaces is needed. These two virtual interfaces come with a virtual cable - what comes at one of the ends goes to the other (like a Linux pipe). So if we want to connect a namespace (let’s say N1) and another one (let’s say N2) we have to put one of the virtual interfaces in the network stack of N1 and the other in the network stack of N2.
![[Pasted image 20250219194619 1.png]]
Let’s build a functional network between the different network namespaces! It’s important to note that there are two types of network namespaces - named and anonymous. The details are not going to be discussed in this article. First we’re going to create a network namespace and then create a pair of virtual interfaces:
![[Pasted image 20250219200112.png]]
Now we have to test the connectivity of the virtual interfaces.
![[Pasted image 20250219200128.png]]
From the snippet above we can see how to create a new network namespace and connect it to the root namespace using a pipe-like connection. The parent namespace retained one of the interfaces, and passed the other one to the child namespace. Anything that enters one of the ends, comes out through the other end, just as a real network connection.
![[Pasted image 20250219200401.png]]
We saw how to isolate, virtualize, and connect Linux network stacks. Having the power of virtualization normally we would like to go further and create a virtual LAN between processes!

### Connecting multiple namespaces (creating a LAN)
To create a virtual LAN another Linux virtualization utility will be used - the bridge. The Linux bridge behaves like a real level 2 (Ethernet) network switch - it forwards packets between interfaces that are connected to it using a MAC association table. Let’s create our virtual LAN.

### Reaching the outside world
https://blog.quarkslab.com/digging-into-linux-namespaces-part-1.html

**To sum up about the network namespace**:
- Processes within a given network namespace get their own private network stack, including network interfaces, routing tables, iptables rules, sockets (ss, netstat);
- The connection between network namespaces can be done using two virtual interfaces;
- Communication between isolated network stacks in the same namespace is done using a bridge;
- The NET namespace can be used to simulate a “box” of Linux processes where only a few of them would be able to reach the outside world (by removing the host’s default gateway from the routing rules of some NET namespaces).

## USER 
All processes in the Linux world have their owner. There are **privileged** and **unprivileged** processes depending on their effective user ID (UID) attribute. Depending on this UID, processes have different privileges over the OS. The _user namespace_ is a kernel feature allowing per-process virtualization of this attribute. In the Linux documentation, a user namespace is defined in the following manner:

>User namespaces isolate security-related identifiers and attributes, in particular, user IDs and group IDs, the root directory, keys, and capabilities. A process’s user and group IDs can be different inside and outside a user namespace. In particular, a process can have a normal unprivileged user ID outside a user namespace while at the same time having a user ID of 0 inside the namespace.

Starting with Linux 3.8 (and unlike the flags used for creating other types of namespaces), on some Linux distributions, no privilege is required to create a user namespace. Let’s try it out!
![[Pasted image 20250219203421.png]]
In the new user namespace our process belongs to user `nobody` with effective UID=65334, which is not present in the system. Okay, but where does it come from and how does the OS resolve it when it comes to system wide operations (modifying files, interacting with programs)? According to the Linux documentation it’s predefined in a file:

>If a user ID has no mapping inside the namespace, then system calls that return user IDs return the value defined in the file /proc/sys/kernel/overflowuid, which on a standard system defaults to the value 65534. Initially, a user namespace has no user ID mapping, so all user IDs inside the namespace map to this value.

To answer the second part of the question - there is dynamic user id mapping when a process needs to perform system-wide operations.
![[Pasted image 20250219203800.png]]

### Mapping UIDs and GIDs
Some processes need to run under effective UID 0 in order to provide their services and be able to interact with the OS file system. One of the most common things when using user namespaces is to define mappings. This is done using the `/proc/<PID>/uid_map` and `/proc/<PID>/gid_map` files. The format is the following:

>ID-inside-ns ID-outside-ns length

_ID-inside-ns (resp.ID-outside-ns)_ defines the starting point of UID mapping inside the user namespace (resp. outside of the user namespace) and length defines the number of subsequent UID (resp. GID) mappings. The mappings are applied when a process within a USER namespace tries to manipulate system resources belonging to another USER namespace.

Some important rules according the Linux documentation:

> If the two processes are in the same namespace, then ID-outside-ns is interpreted as a user ID (group ID) in the parent user namespace of the process PID. The common case here is that a process is writing to its own mapping file (/proc/self/uid_map or /proc/self/gid_map).

> If the two processes are in different namespaces, then ID-outside-ns is interpreted as a user ID (group ID) in the user namespace of the process opening /proc/PID/uid_map (/proc/PID/gid_map). The writing process is then defining the mapping relative to its own user namespace.

![[Pasted image 20250219205547.png]]
The process inside the user namespace thinks his effective UID is root but in the upper (root) namespace his UID is the same as the process that created it (zsh with UID=1000). Here is an illustration of the above snippet.
![[Pasted image 20250219205610 1.png]]
A more general view of the remapping process can be seen on this picture:
![[Pasted image 20250219205655.png]]

Another important thing to note is that **in the defined USER namespace the process has effective UID=0 and all capabilities set in the permitted set.**

Let’s see what is the view of the filesystem for the process within the remapped user namespace.
![[Pasted image 20250219220319.png]]
As it can be seen, the remapped user perceives the ownership of the files with respect to the current user namespace mapping. As the file is owned by UID=1000 which is mapped to UID=0 (root), the process see’s the file as owned by the root user in the current namespace.

### Inspect the current mappings of a process
The `/proc/<PID>/uid_map` and `/proc/<PID>/gid_map` files can also be used for inspecting the mapping of a given process. This inspection is relative to the user namespace a process is within. There are two rules:
- If a process within the same namespace inspects the mapping of another one in the same namespace, it’s going to see the mapping to the parent user namespace.
    
- If a process from another namespace inspects the mapping of a process, it’s going to see a mapping relative to its own mapping.

Okay, we can see that the remapped sh shell process perceives the user remapping based on its own user namespace. The results can be interpreted in the following way: Process UID=0 in the user namespace for process 6638 corresponds to UID=200 in the current namespace. It’s all relative! According to the Linux documentation:

### User namespace and capabilities
the first process inside a new user namespace has the full set of capabilities **inside the current user namespace**. According to the Linux documentation:

>When a user namespace is created, the first process in the namespace is granted a full set of capabilities in the namespace. This allows that process to perform any initializations that are necessary in the namespace before other processes are created in the namespace. Although the new process has a full set of capabilities in the new user namespace, it has no capabilities in the parent namespace. This is true regardless of the credentials and capabilities of the process that calls clone(). In particular, even if root employs a clone(CLONE_NEWUSER), the resulting child process will have no capabilities in the parent namespace.

User Namespace 是 Linux 内核提供的 **用户权限隔离机制**，属于 Linux 命名空间（Namespace）技术栈的一部分。它允许进程在一个隔离的用户权限环境中运行，使容器或沙盒中的用户权限与宿主机完全解耦。

**核心目标**：

- 实现 **UID/GID 虚拟化**（容器内 root ≠ 宿主机 root）
- 支持 **非特权用户创建特权环境**（如无宿主机 root 权限的用户运行容器）
- 增强系统安全性（通过权限边界隔离）

##### 1. UID/GID 映射机制

User Namespace 通过 **映射表** 实现虚拟 ID 到真实 ID 的转换：

- **映射文件**：`/proc/<pid>/uid_map`（用户映射）和 `/proc/<pid>/gid_map`（组映射）
- **映射规则**：`<虚拟ID> <真实ID> <数量>`  
    示例：`0 1000 1` 表示虚拟 UID 0（容器 root）映射到宿主机 UID 1000（普通用户）
##### 2. 特权能力控制

- **Capability 隔离**：在 User Namespace 内部，进程可拥有完整 Capabilities（如 `CAP_SYS_ADMIN`），但这些权限仅在当前 Namespace 内有效。
- **特权操作限制**：跨 Namespace 操作（如挂载全局文件系统）需宿主权限。

##### 3. 父子 Namespace 关系

- **父 Namespace**：可控制子 Namespace 的 UID/GID 分配范围。
- **子 Namespace**：内部进程对映射范围内的 UID/GID 拥有完全控制权，但无法突破映射边界。

- **UID** 是用户的唯一身份标识，决定“你是谁”。
- **GID** 是用户的主组标识，决定“你的默认归属”。
- **Groups** 是权限扩展机制，决定“你还能做什么”。  
    三者共同构建了 Linux 权限体系的基石，理解其差异与协作逻辑，是系统管理、安全加固和容器化部署的核心基础。

## MNT 
Mount (MNT) namespaces are a powerful tool for creating per-process file system trees, thus per-process root filesystem views. Linux maintains a data structure for all the different filesystems mounted on the system. This structure is a per-process attribute and also per-namespace. It includes information like what disk partitions are mounted, where they are mounted and the type of mounting (RO/RW).
![[Pasted image 20250220150311.png]]
Linux namespaces give the possibility to copy this data structure and give the copy to different processes. In that way these processes can alter this structure (mount and unmount) without affecting the mounting points of each other. By providing different copies of the file system structure, the kernel isolates the list of mount points seen by the processes in a namespace. Defining a mount namespace also can allow a process to change its root - a behavior similar to the chroot syscall. The difference is that chroot is bound to the current file system structure and all changes (mounting, unmounting) in the re-rooted environment will affect the entire file system. With the mount namespaces this is not possible as the whole structure is virtualized, thus providing full isolation of the original file system in terms of mount and unmount events.
![[Pasted image 20250221155725.png]]

Here is a general overview of the mount namespace concept.
![[Pasted image 20250221155729.png]]

Let’s try it out!
![[Pasted image 20250221160036.png]]
We can see from the snippet that processes in the isolated mount namespace can create different mount points and files under them without reflecting the parent mount namespace.

### Shared subtrees
Mounting and unmounting directories reflect the OS filesystem. In a classical Linux OS this system is represented as a tree. As the picture above shows, creating different mount namespaces technically results in creating virtual per-process tree structures. These structures can be shared. According to the Linux documentation:

>The key benefit of shared subtrees is to allow automatic, controlled propagation of mount and unmount events between namespaces. This means, for example, that mounting an optical disk in one mount namespace can trigger a mount of that disk in all other namespaces.

The shared trees consist mainly of marking each mount point with a tag saying what’s going to happen if we add/remove mount points which are present in different mount namespaces. The addition/removal of mount points trigger an event which is propagated in the so called peer-groups. A peer group is a set of vfsmounts (virtual file system mount points) that propagate mount and unmount events to one another.

There are 4 types of mountings:

> MS_SHARED: This mount point shares mount and unmount events with other mount points that are members of its “peer group”. When a mount point is added or removed under this mount point, this change will propagate to the peer group, so that the mount or unmount will also take place under each of the peer mount points. Propagation also occurs in the reverse direction, so that mount and unmount events on a peer mount will also propagate to this mount point.

> MS_PRIVATE: This is the converse of a shared mount point. The mount point does not propagate events to any peers, and does not receive propagation events from any peers.

> MS_SLAVE: This propagation type sits midway between shared and private. A slave mount has a master — a shared peer group whose members propagate mount and unmount events to the slave mount. However, the slave mount does not propagate events to the master peer group.

> MS_UNBINDABLE - Does not receive or forward any propagation events and cannot be bind mounted.

The mount point state is per mount point. This means that if you have / and /boot, for example, you’d have to separately apply the desired state to each mount point. Let’s play a little bit with all that!
![[Pasted image 20250221160755.png]]


## UTS namespace
UTS namespace isolates the system hostname for a given process.

> Most communication to and from a host is done via the IP address and port number. However, it makes life a lot easier for us humans when we have some sort of name attached to a process. Searching through log files, for instance, is much easier when identifying a hostname. Not the least of which because, in a dynamic environment, IPs can change.

> In our building analogy, the hostname is similar to the name of the apartment building. Telling the taxi driver I live in City Place apartments is usually more effective than providing the actual address. Having multiple hostnames on a single physical host is a great help in large containerized environments.

So the UTS namespace provides hostname segregation for processes. In that way, it’s easier to communicate with services on a private network, and to inspect their logs on the host.

## IPC namespace
The IPC namespace provides isolation for process communication mechanisms such as semaphores, message queues, shared memory segments, etc. Normally when a process is forked it inherits all the IPC’s which were opened by its parent. The processes inside an IPC namespace can’t see or interact with the IPC resources of the upper namespace. Here is a brief example using shared memory segments.
![[Pasted image 20250221161025.png]]


## CGROUP namespace
Cgroups is a technology which controls the amount of hardware resources (RAM, HDD, block I/O) consumed by a process.

>By default, CGroups are created in the virtual filesystem /sys/fs/cgroup. Creating a different CGroup namespace essentially moves the root directory of the CGroup. If the CGroup was, for example, /sys/fs/cgroup/mycgroup, a new namespace CGroup could use this as a root directory. The host might see /sys/fs/cgroup/mycgroup/{group1,group2,group3} but creating a new CGroup namespace would mean that the new namespace would only see {group1,group2,group3}.

So the Cgroup namespaces virtualize another virtual filesystem as the PID namespace. But what’s the purpose of providing isolation for this system? According to the man page:

> It prevents information leaks whereby cgroup directory paths outside of a container would otherwise be visible to processes in the container. Such leakages could, for example, reveal information about the container framework to containerized applications.

> In a traditional CGroup hierarchy, there is a possibility that a nested CGroup could gain access to its ancestor. This means that a process in /sys/fs/cgroup/mycgroup/group1 has the potential to read and/or manipulate anything nested under mycgroup.

# Combine almost everything
Let’s use everything we discussed to build a fully isolated environment for a given process, step by step, using the unshare wrapper! A POC of this article written in C and using the _clone()_ syscall is also available [here](https://github.com/mihailkirov/namespaces/tree/main/PoC).

It’s important to note the order of the namespace wrapping procedure as some operations (such as PID, UTC, IPC namespace creation) need extended privileges in the current namespace. The following algorithm illustrates an order which would do the job.
![[Pasted image 20250220222315.png]]

这段代码使用了 `clone` 系统调用来创建一个新的进程，具体解释如下：

![[Pasted image 20250220231631.png]]


### 详细解释：

#### 1. `clone` 函数：

`clone` 是一个 Linux 系统调用，用来创建一个新的进程或线程。与 `fork()` 类似，`clone` 允许你指定进程或线程共享哪些资源，如内存空间、文件描述符等。它比 `fork()` 更加灵活，能够实现更细粒度的控制。

- `cmd_exec`：这是子进程执行的函数（子进程的入口点）。这个函数会在子进程启动时执行。
- `stack_child + STACK_SIZE`：这是子进程的栈地址。假设子进程的栈从 `stack_child` 开始，栈的大小为 `STACK_SIZE`，`stack_child + STACK_SIZE` 表示栈的顶端（因为栈是向下增长的）。
- `flags`：指定了 `clone` 调用的行为。这里使用了多个标志来定义进程的属性。

#### 2. `flags` 参数：

`flags` 控制了子进程的资源和行为。这个例子中，使用了多个标志来影响子进程的创建方式：

- `SIGCHLD`：当子进程终止时，父进程会收到 `SIGCHLD` 信号。
- `CLONE_NEWUSER`：创建一个新的用户命名空间（隔离用户ID）。
- `CLONE_NEWNET`：创建一个新的网络命名空间（隔离网络资源）。
- `CLONE_NEWUTS`：创建一个新的 UTS 命名空间（隔离主机名和域名）。
- `CLONE_NEWIPC`：创建一个新的 IPC 命名空间（隔离进程间通信）。
- `CLONE_NEWNS`：创建一个新的挂载命名空间（隔离挂载点）。
- `CLONE_NEWPID`：创建一个新的 PID 命名空间（隔离进程 ID）。
- `CLONE_NEWCGROUP`：创建一个新的 cgroup 命名空间（隔离资源控制组）。

通过这些标志，子进程会在多个方面与父进程隔离，例如独立的 PID、网络、用户等。

#### 3. `stack_child + STACK_SIZE`：

- `stack_child` 是父进程分配给子进程的栈空间的起始地址，`STACK_SIZE` 是栈的大小。`stack_child + STACK_SIZE` 指向栈的顶部，因为栈是向下增长的。

#### 4. `c`：

- `c` 是传递给子进程的参数（通常是结构体或指针）。这个参数通常用于向子进程传递一些数据。

#### 5. 错误检查：

- 如果 `clone` 调用返回 `-1`，说明进程创建失败。此时，`strerror(errno)` 会返回一个描述错误的字符串，`fprintf` 会将错误信息输出到标准错误流，并且程序会调用 `exit(EXIT_FAILURE)` 退出。

### 总结：

- 该代码使用 `clone` 创建了一个新的进程，并且通过不同的命名空间标志将子进程与父进程隔离。
- 子进程会执行 `cmd_exec` 函数，并且会使用 `stack_child + STACK_SIZE` 作为栈地址。
- 通过设置 `SIGCHLD` 标志，父进程能够在子进程结束时收到信号通知。
- 如果创建子进程失败，会输出错误信息并退出程序。

`clone` 是非常底层的系统调用，通常用于实现容器技术、虚拟化等高级功能。


### **代码功能解析：禁用 `setgroups` 以实现用户命名空间组映射**

#### **1. 代码逻辑分解**
![[Pasted image 20250220233924.png]]

#### **2. 核心目标**

- **禁用 `setgroups` 系统调用**：  
    在 Linux **User Namespace** 中，当需要为进程配置组 ID（GID）映射（`gid_map`）时，必须先禁用 `setgroups` 系统调用。
    - `setgroups` 允许进程动态修改其**附加组列表**（Supplementary Groups），可能绕过命名空间的权限隔离。
    - 内核强制要求：**写入 `gid_map` 前必须锁定 `setgroups` 策略**，否则映射操作会被拒绝。

#### **3. 关键文件与机制**

- **`/proc/[pid]/setgroups`**：
    - 控制当前进程所属命名空间的 `setgroups` 权限，仅接受两个值：
        - **`deny`**：禁止调用 `setgroups`，且必须在该操作后才能写入 `gid_map`。
        - **`allow`**（默认）：允许调用 `setgroups`，但写入 `gid_map` 会被拒绝。
    - **安全意义**：防止恶意进程通过修改附加组提升权限，破坏命名空间隔离性。

#### **4. 典型应用场景**

- **容器启动流程**（如 Docker、LXC）：  
    创建容器时，父进程需为新命名空间内的进程配置 UID/GID 映射，禁用 `setgroups` 是必要步骤。
- **沙盒环境隔离**：  
    确保子进程无法通过组权限修改突破资源访问限制。

#### **5. 操作顺序要求**
1. **写入 `setgroups`**：设为 `deny` 以禁用动态组修改。
2. **配置 `gid_map`**：定义虚拟 GID 到宿主机真实 GID 的映射。
3. **失败条件**：
    - 若未禁用 `setgroups` 直接写 `gid_map`，内核返回 `EPERM` 错误。
    - 若尝试在 `setgroups` 写入后重新启用（改为 `allow`），操作会被拒绝。

### **安全与兼容性**

- **内核版本要求**：
    - Linux 3.19+ 强制要求此操作，旧版本可能无需显式禁用 `setgroups`。
- **特权需求**：
    - 修改 `setgroups` 和 `gid_map` 需具备 `CAP_SYS_ADMIN` 能力（通常在命名空间创建时授予）。
- **容器逃逸防御**：  
    禁用 `setgroups` 阻止攻击者通过添加高危组（如 `docker` 或 `sudo`）获取宿主权限。

### 一、查看 Linux 命名空间的核心方法（5 类场景）

---

#### **1. 列出所有命名空间类型（系统级）**

Linux 内核默认支持的命名空间类型包括：

- **PID**：隔离进程树
- **Mount (mnt)**：隔离文件系统挂载点
- **Network (net)**：隔离网络设备/IP/路由表
- **UTS**：隔离主机名和域名
- **IPC**：隔离进程间通信资源
- **User**：隔离用户和用户组 ID
- **Cgroup**：隔离 cgroup 根目录
- **Time**：隔离系统时钟（内核 5.6+）

**查看系统支持的类型**：
```
cat /proc/self/ns/* # 查看当前进程的所有命名空间类型
```

#### **2. 列出所有存在的命名空间实例（工具法）**

使用 **`lsns`** 命令（需安装 `util-linux` 包）快速列出所有命名空间实例：
```
sudo lsns -t net # 查看所有网络命名空间（示例）
```

输出示例：

![[Pasted image 20250221112237.png]]

- **关键字段**：
    - `NS`：命名空间唯一标识（inode 编号）
    - `TYPE`：命名空间类型
    - `NPROCS`：关联的进程数
    - `NETNSID`：网络命名空间 ID（仅 net 类型有效）

