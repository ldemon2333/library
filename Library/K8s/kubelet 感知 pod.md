### Kubelet 感知 Pod 并准备拉取镜像

当 Pod 被调度器绑定到 Kubelet 所在的节点后，Kubelet 就会感知到这个 Pod，并开始接管其生命周期。

1. **Kubelet 感知 Pod：** Kubelet 持续 watch API Server。当它发现一个 Pod 对象的 `spec.nodeName` 字段与自己的节点名匹配时，Kubelet 就知道这个 Pod 被调度到了自己这个节点。
    
2. **Pod Admission & Validation (Kubelet 内部)：** Kubelet 会对 Pod 的配置进行一些最终的本地验证和准入检查。
    
3. **Pod 状态更新：** Kubelet 会将该 Pod 的状态从 `Pending` 更新为 `ContainerCreating`。这个状态表明 Kubelet 已经开始准备 Pod 的运行环境。
    
4. **Pod 沙箱准备 (Pod Sandbox Creation)：**
    
    - **请求容器运行时：** Kubelet 通过 **CRI (Container Runtime Interface)** 调用节点上的容器运行时（如 containerd 或 CRI-O）。
        
    - **创建 Pod 沙箱 (Pod Sandbox)：** 容器运行时会创建一个 Pod 沙箱，这通常对应一个 Pause 容器（或 Infra 容器）。这个 Pause 容器持有 Pod 的网络命名空间、IPC 命名空间等共享资源。它是 Pod 中所有其他容器的“宿主”。
        
    - **分配 IP 地址：** 在沙箱创建过程中，如果配置了 CNI (Container Network Interface) 插件，CNI 会为 Pod 分配一个 IP 地址并设置网络。
        
5. **容器清单解析和准备：** Kubelet 会解析 Pod 定义中每个容器的镜像信息、环境变量、卷挂载等。
    
6. **决定镜像拉取策略：** 对于 Pod 中的每个容器，Kubelet 会根据容器的 `imagePullPolicy` 决定是否需要拉取镜像：
    
    - **`Always` (默认用于 `:latest` 标签或未指定标签的镜像)：** 每次启动容器前都尝试从镜像仓库拉取最新镜像。
        
    - **`IfNotPresent` (默认用于非 `:latest` 标签的镜像)：** 只有当本地没有该镜像时才拉取。
        
    - **`Never`：** 永不拉取，只使用本地镜像（如果不存在则容器会启动失败）。
        
7. **检查本地镜像缓存：** 如果 `imagePullPolicy` 允许使用本地镜像，Kubelet 会首先检查节点上的容器运行时是否已经缓存了该镜像。
    

---

### 开始拉取镜像

如果 Kubelet 决定需要拉取镜像（因为本地没有或者策略是 `Always`），它就会执行以下操作：

1. **请求容器运行时拉取镜像：** Kubelet 通过 CRI 调用容器运行时（如 containerd），指示其拉取指定的容器镜像。
    
2. **镜像拉取：** 容器运行时会根据镜像的名称、标签和可能包含的私有仓库认证信息（通常来自 Secret）去配置的镜像仓库拉取镜像。
    
3. **镜像拉取进度：** 在此期间，Pod 的状态可能会继续保持 `ContainerCreating`，或者在某些情况下，你可能会在 Pod 事件中看到 `Pulling image` 或 `ImagePullBackOff`（如果拉取失败）。
    

---

### 总结

从节点 `Ready` 到 Kubelet 开始拉取 Pod 镜像，Kubelet 经历了从**监听调度结果**、**Pod 本地准入验证**、**准备 Pod 沙箱（包括网络）**，到**解析容器定义**、**评估镜像拉取策略**，并最终**请求容器运行时拉取镜像**的整个流程。这是一个多步骤、精确协调的过程，确保 Pod 能够按照预期在节点上启动。


在 Kubelet 内部，负责“感知 Pod”的核心机制是其与 **Kubernetes API Server** 的持续交互，特别是通过 **watch 机制**。并没有一个单独的“感知 Pod 组件”模块，而是 Kubelet 的主循环和其内部的控制器（Controller）协同完成这一任务。

---

### 核心机制：API Server Watcher

Kubelet 启动后，它会执行以下关键步骤来“感知”分配给它的 Pod：

1. **节点注册：** Kubelet 首先向 API Server 注册自己作为一个节点，并将其状态报告为 `Ready`。
    
2. **Pod 监听器/同步器 (Pod Listener/Syncer)：** Kubelet 内部有一个核心的**同步循环 (sync loop)** 或称为**主循环**，这个循环持续地从 API Server 获取并同步其管辖的 Pod 状态。它主要通过以下两种方式：
    
    - **Watch API (推荐和常用):** Kubelet 会打开一个与 API Server 的持久连接，并对分配给其节点的 **Pod 对象**进行 **watch 操作**。当 Pod 被 **kube-scheduler** 调度到 Kubelet 所在的节点上时（即 Pod 的 `spec.nodeName` 字段被更新为该节点名称），API Server 会立即将这个事件通知给 Kubelet。这是 Kubelet 感知新 Pod 的主要和实时的方式。
        
    - **List-and-Watch 机制：** 在 Kubelet 启动时或连接中断后，它会执行一个 `List` 操作，获取所有分配给自己的 Pod 的当前完整列表，然后切换到 `Watch` 模式以接收后续的增量更新。这种“先列出再观察”的机制确保了即使在网络抖动或 Kubelet 重启后也能保持状态一致。
        
3. **Pod 管理器 (Pod Manager):** Kubelet 内部有一个组件负责管理 Pod 的本地存储和状态。当通过 watch 机制感知到新的 Pod 时，Kubelet 会将 Pod 定义存储在内存中，并将其加入到待处理队列。
    

---

### Kubelet 内部处理 Pod 的流程（简化）

当 Kubelet 通过 watch 机制接收到一个新 Pod 对象（`spec.nodeName` 匹配本节点）的通知后，它会：

1. **更新本地 Pod 状态：** Kubelet 会在自己的内部数据结构中更新这个 Pod 的状态。
    
2. **处理 Pod 生命周期：** Kubelet 的主循环会根据这个 Pod 的当前状态和期望状态，触发一系列动作，例如：
    
    - **Pod Admission/Validation：** 对 Pod 的配置进行一些本地的准入和验证。
        
    - **Pod 沙箱创建：** 调用容器运行时接口 (CRI) 创建 Pod 沙箱（通常是 Pause 容器），并设置网络（通过 CNI 插件）。
        
    - **镜像拉取：** 根据容器的 `imagePullPolicy` 检查本地镜像或向容器运行时发出拉取镜像的请求。
        
    - **容器创建和启动：** 指示容器运行时创建并启动 Pod 中定义的业务容器。
        
    - **探针检查：** 启动并定期执行 Pod 的存活（Liveness）和就绪（Readiness）探针。
        
    - **状态报告：** 持续将 Pod 的实际运行状态（例如 `Pending` -> `ContainerCreating` -> `Running`）报告给 API Server。
        

---

### 总结

虽然没有一个明确命名的“感知 Pod”组件，但这项功能的核心是由 **Kubelet 内部的 API Server 客户端**（通过 watch 机制）以及其 **主循环/Pod 同步器** 来完成的。这些组件持续地监视 API Server 中 Node 对象上分配给自己的 Pod 定义，一旦发现有新的 Pod 被调度到本节点，便立即接管其生命周期管理。