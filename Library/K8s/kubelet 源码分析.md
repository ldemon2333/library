执行 `kubelet -h` 看到 kubelet 的功能介绍：

- kubelet 是每个 Node 节点上都运行的主要“节点代理”。使用如下的一个向 apiserver 注册 Node 节点：主机的 `hostname`；覆盖 `host` 的参数；或者云提供商指定的逻辑。
- kubelet 基于 `PodSpec` 工作。`PodSpec` 是用 `YAML` 或者 `JSON` 对象来描述 Pod。Kubelet 接受通过各种机制（主要是 apiserver）提供的一组 `PodSpec`，并确保里面描述的容器良好运行。

除了由 apiserver 提供 `PodSpec`，还可以通过以下方式提供：

- 文件
- HTTP 端点
- HTTP 服务器

kubelet 功能归纳一下就是上报 Node 节点信息，和管理（创建、销毁）Pod。 功能看似简单，实际不然。每一个点拿出来都需要很大的篇幅来讲，比如 Node 节点的计算资源，除了传统的 CPU、内存、硬盘，还提供扩展来支持类似 GPU 等资源；Pod 不仅仅有容器，还有相关的网络、安全策略等。

# 架构
![[Pasted image 20250709100320.png]]
## PLEG
即 **Pod Lifecycle Event Generator**，字面意思 Pod 生命周期事件（`ContainerStarted`、`ContainerDied`、`ContainerRemoved`、`ContainerChanged`）生成器。

其维护着 Pod 缓存；定期通过 `ContainerRuntime` 获取 Pod 的信息，与缓存中的信息比较，生成如上的事件；将事件写入其维护的通道（channel）中。




## PodWorkers
处理事件中 Pod 的同步。核心方法 `managePodLoop()` 间接调用 `kubelet.syncPod()` 完成 Pod 的同步：

- 如果 Pod 正在被创建，记录其延迟
- 生成 Pod 的 API Status，即 `v1.PodStatus`：从运行时的 status 转换成 api status
- 记录 Pod 从 `pending` 到 `running` 的耗时
- 在 `StatusManager` 中更新 pod 的状态
- 杀掉不应该运行的 Pod
- 如果网络插件未就绪，只启动使用了主机网络（host network）的 Pod
- 如果 static pod 不存在，为其创建镜像（Mirror）Pod
- 为 Pod 创建文件系统目录：Pod 目录、卷目录、插件目录
- 使用 `VolumeManager` 为 Pod 挂载卷
- 获取 image pull secrets
- 调用容器运行时（container runtime）的 `#SyncPod()` 方法

## PodManager
存储 Pod 的期望状态，kubelet 服务的不同渠道的 Pod

## StatsProvider
提供节点和容器的统计信息，有 `cAdvisor` 和 `CRI` 两种实现。

## Deps.PodConfig
PodConfig 是一个配置多路复用器，它将许多 Pod 配置源合并成一个单一的一致结构，然后按顺序向监听器传递增量变更通知。

配置源有：文件、apiserver、HTTP

## syncLoop
接收来自 `PodConfig` 的 Pod 变更通知、定时任务、`PLEG` 的事件，以及 `ProbeManager` 的事件，将 Pod 同步到**期望状态**。

syncLoop is the main loop for processing changes. It watches for changes from three channels (**file, apiserver, and http***) and creates a union of them. For any new change seen, will run a sync against desired state and running state. If no changes are seen to the configuration, will synchronize the last known desired state every sync-frequency seconds. **Never returns**. Kubelet启动后通过syncLoop进入到主循环处理Node上Pod Changes事件，监听来自file,apiserver,http三类的事件并汇聚到kubetypes.PodUpdate Channel（Config Channel）中，由syncLoopIteration不断从kubetypes.PodUpdate Channel中消费。

![[Pasted image 20250709103625.png]]

## PodAdmitHandlers
Pod admission 过程中调用的一系列处理器，比如 eviction handler（节点内存有压力时，不会驱逐 QoS 设置为 `BestEffort` 的 Pod）、shutdown admit handler（当节点关闭时，不处理 pod 的同步操作）等。

## OOMWatcher
从系统日志中获取容器的 OOM 日志，将其封装成事件并记录。

## VolumeManager
VolumeManager 运行一组异步循环，根据在此节点上调度的 pod 确定需要附加/挂载/卸载/分离哪些卷并执行操作。

## CertificateManager
处理证书轮换。

## ProbeManager
实际上包含了三种 Probe，提供 probe 结果缓存和通道。

- LivenessManager
- ReadinessManager
- StartupManager

## EvictionManager
监控 Node 节点的资源占用情况，根据驱逐规则驱逐 Pod 释放资源，缓解节点的压力。

## PluginManager
PluginManager 运行一组异步循环，根据此节点确定哪些插件需要注册/取消注册并执行。如 CSI 驱动和设备管理器插件（Device Plugin）。

# 源码分析
从 `git@github.com:kubernetes/kubernetes.git` 仓库获取代码，使用最新的 `release-1.21` 分支。

- `cmd/kubelet/kubelet.go:35` 的 `main` 方法为程序入口。
    - 调用 `NewKubeletCommand` 方法，创建 command
    - 执行 command
        - `cmd/kubelet/app/server.go:434` 的 `Run` 方法。
            - 调用 `RunKubelet` 方法。
                - 调用 `createAndInitKubelet` 方法，创建并初始化 kubelet
                    - `pkg/kubelet/kubelet.go` 的 `NewMainKubelet` 方法，创建 kubelet的 各种组件。共十几个组件，见 [kubelet 的构架](https://atbug.com/kubelet-source-code-analysis/#kubelet-%E6%9E%B6%E6%9E%84)。
                    - 调用 `BirtyCry` 方法：放出 `Starting` 事件
                    - 调用 `StartGarbageCollection` 方法，开启 `ContainerGC` 和 `ImageGC`
                - 调用 `startKubelet` 方法（大量使用 goroutine 和通道）
                    - goroutine：`kubelet.Run()`
                        - 初始化模块
                            - metrics 相关
                            - 创建文件系统目录目录
                            - 创建容器日志目录
                            - 启动 `ImageGCManager`
                            - 启动 `ServerCertificateManager`
                            - 启动 `OOMWatcher`
                            - 启动 `ResourceAnalyzer`
                        - goroutine：`VolumeManager.Run()` 开始处理 Pod Volume 的卸载和挂载
                        - goroutine：状态更新 `fastStatusUpdateOnce()` （更新 Pod CIDR -> 更新 `ContainerRuntime` 状态 -> 更新 Node 节点状态）
                        - goroutine： `NodeLeaseController.Run()` 更新节点租约
                        - goroutine：`podKiller.PerformPodKillingWork` 杀掉未被正确处理的 pod
                        - `StatusManager.Start()` 开始向 apiserver 更新 Pod 状态
                        - `RuntimeClassManager.Start()`
                        - `PLEG.Start()`：持续从 `ContainerRuntime` 获取 Pod/容器的状态，并与 kubelet 本地 cache 中的比较，生成对应的 `Event`
                        - `syncLoop()` 重点，**_持续监控并处理来自文件、apiserver、http 的变更_**。包括 Pod 的增加、更新、优雅删除、非优雅删除、调和。
            - 启动 server，暴露 `/healthz` 端点
            - 通知 `systemd` `kuberlet` 服务已经启动


---

## PLEG 如何通过 CRI 与容器运行时通信

PLEG（Pod 生命周期事件生成器）与底层**容器运行时接口 (CRI)** 的通信是 Kubelet 能够感知 Pod 和容器状态变化的核心机制。这个过程并非简单的“对话”，而是一系列标准的 API 调用和响应。

为了详细展开这个过程，我们先了解一下 CRI 的构成。CRI 定义了两组主要的 gRPC 服务：

1. **ImageService (镜像服务):** 负责镜像的生命周期管理，例如拉取、删除、列出镜像等。
    
2. **RuntimeService (运行时服务):** 负责 Pod 沙箱和容器的生命周期管理，例如创建沙箱、启动容器、停止容器、获取容器状态等。
    

PLEG 主要与 **RuntimeService** 进行交互。下面是 PLEG 与 CRI 进行通信的详细过程：

### 1. Kubelet 启动并建立 CRI 连接

当 Kubelet 启动时，它会初始化其内部的 CRI 客户端。这个客户端会通过 gRPC 连接到节点上运行的**容器运行时**（例如 containerd 或 CRI-O）。每个容器运行时都会暴露一个符合 CRI 规范的 gRPC 服务端点（通常是 Unix socket）。

### 2. PLEG 的轮询循环

PLEG 内部有一个持续运行的**轮询（relist）循环**。这个循环会以一个预设的间隔（例如 Kubelet 配置中的 `pleg-relist-period`，默认为 10 秒）被触发。每次触发时，PLEG 就会开始向容器运行时发起一系列的 CRI 调用。

### 3. PLEG 发送 `ListPodSandbox` 请求

在每次轮询开始时，PLEG 会向容器运行时服务的 `RuntimeService` 发送一个 **`ListPodSandbox`** gRPC 请求。

- **目的：** 这个请求的目的是获取当前节点上所有**Pod 沙箱 (Pod Sandbox)** 的列表及其最新的状态。Pod 沙箱是 Pod 在容器运行时中的抽象，包含了 Pod 共享的网络、PID 和 IPC 命名空间，以及 Pod 的基础元数据。
    
- **请求内容：** 通常，这个请求是空的或者只包含一些过滤条件（如果需要）。
    
- **容器运行时响应：** 容器运行时接收到请求后，会查询其内部状态，并将节点上所有活跃的 Pod 沙箱的 **`PodSandboxStatus`** 列表作为响应返回给 PLEG。每个 `PodSandboxStatus` 包含了沙箱的 ID、状态（`Ready` 或 `NotReady`）、创建时间、网络信息（如 Pod IP）以及关联的 Pod 元数据等。
    

### 4. PLEG 发送 `ListContainers` 请求

紧接着，PLEG 会向 `RuntimeService` 发送一个或多个 **`ListContainers`** gRPC 请求。

- **目的：** 这个请求是为了获取所有运行中容器的详细状态。
    
- **请求内容：** 通常会包含一些过滤器，例如只列出属于特定 Pod 沙箱的容器，或者只列出处于特定状态的容器。
    
- **容器运行时响应：** 容器运行时会返回一个 **`ContainerStatus`** 列表。每个 `ContainerStatus` 对象包含了容器的 ID、名称、镜像、当前的运行状态（`Running`、`Waiting` 或 `Terminated`）、重启次数以及更详细的 `ContainerState` 信息（如退出码、开始/结束时间等）。
    

### 5. PLEG 进行状态对比和事件生成

这是 PLEG 的核心逻辑。一旦 PLEG 从容器运行时获取到最新的 Pod 沙箱和容器状态列表，它会执行以下操作：

- **内部状态缓存：** PLEG 内部会维护一个数据结构，存储上一次轮询时获取到的所有 Pod 和容器的状态快照。
    
- **逐一对比：** PLEG 会将当前轮询获取到的新状态与上一次缓存的状态进行逐一对比。
    
    - 如果发现一个新的 Pod 沙箱或容器（之前不存在）。
        
    - 如果发现一个现有的 Pod 沙箱或容器的状态发生了变化（例如，从 `Running` 变为 `Terminated`，或者 `restartCount` 增加了）。
        
    - 如果发现一个 Pod 沙箱或容器已经消失。
        
- **生成事件：** 对于每一个检测到的状态变化，PLEG 都会生成一个或多个**Pod 生命周期事件**。这些事件包含了变化前后的状态信息、受影响的 Pod/容器 ID、事件类型（如 `ContainerDied`、`PodSandboxChanged` 等）。
    

### 6. 将事件推送到 Kubelet 事件队列

生成的所有事件都会被推送到 Kubelet 内部的一个**事件队列**（通常是一个 Channel 或类似的并发安全结构）中。

### 7. Kubelet 其他组件消费事件

Kubelet 的其他组件，如 **Pod 同步循环 (Sync Pod Loop)** 和 **探针管理器 (Prober Manager)**，会持续监听并从这个事件队列中消费事件。

- 当它们收到事件时，会根据事件的类型触发相应的逻辑。例如：
    
    - 收到 `ContainerDied` 事件：Kubelet 可能会触发容器的重启逻辑（取决于 Pod 的 `restartPolicy`）。
        
    - 收到 `PodSandboxChanged` 事件：Kubelet 可能会重新同步 Pod 的状态，并更新 API Server 中的 Pod 状态。
        
    - 收到 `ContainerRunning` 事件：探针管理器可能会开始执行就绪/存活探针。
        

---

**总结来说，PLEG 和 CRI 之间的通信是一个周期性的、请求-响应式的过程。** PLEG 主动向容器运行时查询状态（通过 `ListPodSandbox` 和 `ListContainers`），容器运行时则根据 CRI 规范返回详细的状态信息。通过这种持续的轮询和对比机制，PLEG 确保了 Kubelet 始终能够准确、及时地了解节点上 Pod 和容器的真实运行状况，从而做出正确的调度和管理决策。