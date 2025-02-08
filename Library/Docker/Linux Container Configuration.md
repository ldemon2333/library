本文档描述了容器配置中与Linux相关的部分的模式（schema）。Linux容器规范利用多种内核功能来实现容器的隔离和资源管理，这些功能包括：

1. **Namespaces（命名空间）**：命名空间用于隔离不同容器的资源视图。每个命名空间提供了一种独立的系统资源视图，例如进程ID、网络接口、挂载点等。通过命名空间，容器可以拥有独立的进程树、网络栈、用户ID等，从而实现进程间的隔离。

2. **Cgroups（控制组）**：Cgroups用于限制、记录和隔离容器使用的资源（如CPU、内存、磁盘I/O等）。通过Cgroups，可以为容器分配特定的资源配额，并监控其资源使用情况，确保容器不会占用过多的系统资源。

3. **Capabilities（能力）**：Linux内核的能力机制允许对进程的特权进行更细粒度的控制。传统的Unix权限模型是基于“全有或全无”的root权限，而Capabilities允许将root权限分解为多个独立的能力。容器可以通过Capabilities来限制其内部的进程能够执行的特权操作，从而增强安全性。

4. **LSM（Linux安全模块）**：LSM是Linux内核的安全框架，支持多种安全模块，如SELinux、AppArmor等。这些模块可以为容器提供额外的安全策略，限制容器内的进程能够访问的资源，防止容器内的恶意行为影响宿主机或其他容器。

5. **Filesystem Jails（文件系统监狱）**：文件系统监狱通过限制容器对文件系统的访问，确保容器只能访问特定的目录和文件。通常，容器会使用一个独立的根文件系统（rootfs），使得容器内的进程只能看到和操作这个根文件系统中的内容，从而实现文件系统的隔离。

通过这些内核功能，Linux容器规范能够实现容器的隔离、资源控制和安全性，确保容器内的应用程序能够在独立的环境中运行，而不会干扰宿主机或其他容器。

The following filesystems SHOULD be made available in each container's filesystem:

在Linux环境中，应用程序通常依赖于特定的文件路径和文件系统来正常运行。为了确保容器内的应用程序能够像在标准的Linux环境中一样工作，容器文件系统中**应该**包含以下文件系统或文件路径：

1. **`/proc`**  
   - `/proc` 是一个虚拟文件系统，提供了关于系统进程和内核状态的实时信息。许多工具和应用程序依赖 `/proc` 来获取系统信息（如CPU、内存使用情况等）。
   - 例如，`/proc/self` 提供了当前进程的信息，`/proc/cpuinfo` 提供了CPU的详细信息。

2. **`/sys`**  
   - `/sys` 是另一个虚拟文件系统，用于暴露内核设备和驱动的信息。它允许用户空间程序与内核进行交互，配置硬件设备或获取设备状态。
   - 例如，容器内的应用程序可能需要通过 `/sys/class/net/` 访问网络接口信息。

3. **`/dev/pts`**
4. `/dev/shm`

# Namespaces
A namespaces wraps a global system resource in an abstraction that makes it appear to the processes within the namespace that they have their own isolated instance of the global resource.

Namespaces are specified as an array of entries inside the `namespaces` root field.
- **`type`** _(string, REQUIRED)_ - namespace type. The following namespace types SHOULD be supported:
    
    - **`pid`** processes inside the container will only be able to see other processes inside the same container or inside the same pid namespace.
    - **`network`** the container will have its own network stack.
    - **`mount`** the container will have an isolated mount table.
    - **`ipc`** processes inside the container will only be able to communicate to other processes inside the same container via system level IPC.
    - **`uts`** the container will be able to have its own hostname and domain name.
    - **`user`** the container will be able to remap user and group IDs from the host to local users and groups within the container.
    - **`cgroup`** the container will have an isolated view of the cgroup hierarchy.

- **`path`** _(string, OPTIONAL)_ - namespace file. This value MUST be an absolute path in the [runtime mount namespace](https://github.com/opencontainers/runtime-spec/blob/20a2d9782986ec2a7e0812ebe1515f73736c6a0c/glossary.md#runtime-namespace). The runtime MUST place the container process in the namespace associated with that `path`. The runtime MUST [generate an error](https://github.com/opencontainers/runtime-spec/blob/20a2d9782986ec2a7e0812ebe1515f73736c6a0c/runtime.md#errors) if `path` is not associated with a namespace of type `type`.

If a namespace type is not specified in the `namespaces` array, the container MUST inherit the [runtime namespace](https://github.com/opencontainers/runtime-spec/blob/20a2d9782986ec2a7e0812ebe1515f73736c6a0c/glossary.md#runtime-namespace) of that type. If a `namespaces` field contains duplicated namespaces with same `type`, the runtime MUST [generate an error](https://github.com/opencontainers/runtime-spec/blob/20a2d9782986ec2a7e0812ebe1515f73736c6a0c/runtime.md#errors).

```json
"namespaces": [
    {
        "type": "pid",
        "path": "/proc/1234/ns/pid"
    },
    {
        "type": "network",
        "path": "/var/run/netns/neta"
    },
    {
        "type": "mount"
    },
    {
        "type": "ipc"
    },
    {
        "type": "uts"
    },
    {
        "type": "user"
    },
    {
        "type": "cgroup"
    }
]
```


