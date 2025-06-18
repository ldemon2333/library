https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/
官方文档


对 GPU 等硬件加速设备管理的支持

而对于云的用户来说，在 GPU 的支持上，他们最基本的诉求其实非常简单：我只要在 Pod 的 YAML 里面，声明某容器需要的 GPU 个数，那么 Kubernetes 为我创建的容器里就应该出现对应的 GPU 设备，以及它对应的驱动目录。

以 NVIDIA 的 GPU 设备为例，上面的需求就意味着当用户的容器被创建之后，这个容器里必须出现如下两部分设备和目录：

1. GPU 设备，比如 /dev/nvidia0；
    
2. GPU 驱动目录，比如 /usr/local/nvidia/* 。

其中，GPU 设备路径，正是该容器启动时的 Devices 参数；而驱动目录，则是该容器启动时的 Volume 参数。所以，在 Kubernetes 的 GPU 支持的实现里，kubelet 实际上就是将上述两部分内容，设置在了创建该容器的 CRI （Container Runtime Interface）参数里面。这样，等到该容器启动之后，对应的容器里就会出现 GPU 设备和驱动的路径了。

不过，Kubernetes 在 Pod 的 API 对象里，并没有为 GPU 专门设置一个资源类型字段，而是使用了一种叫作 Extended Resource（ER）的特殊字段来负责传递 GPU 的信息。比如下面这个例子：

![[Pasted image 20250527215646.png]]

当然，在 Kubernetes 的 GPU 支持方案里，你并不需要真正去做上述关于 Extended Resource 的这些操作。在 Kubernetes 中，对所有硬件加速设备进行管理的功能，都是由一种叫作 Device Plugin 的插件来负责的。这其中，当然也就包括了对该硬件的 Extended Resource 进行汇报的逻辑。

Kubernetes 的 Device Plugin 机制，我可以用如下所示的一幅示意图来和你解释清楚。
![[Pasted image 20250527215754.png]]

首先，对于每一种硬件设备，都需要有它所对应的 Device Plugin 进行管理，这些 Device Plugin，都通过 gRPC 的方式，同 kubelet 连接起来。以 NVIDIA GPU 为例，它对应的插件叫作[`NVIDIA GPU device plugin`](https://github.com/NVIDIA/k8s-device-plugin)。

这个 Device Plugin 会通过一个叫作 ListAndWatch 的 API，定期向 kubelet 汇报该 Node 上 GPU 的列表。比如，在我们的例子里，一共有三个 GPU（GPU0、GPU1 和 GPU2）。这样，kubelet 在拿到这个列表之后，就可以直接在它向 APIServer 发送的心跳里，以 Extended Resource 的方式，加上这些 GPU 的数量，比如`nvidia.com/gpu=3`。所以说，用户在这里是不需要关心 GPU 信息向上的汇报流程的。

需要注意的是，ListAndWatch 向上汇报的信息，只有本机上 GPU 的 ID 列表，而不会有任何关于 GPU 设备本身的信息。而且 kubelet 在向 API Server 汇报的时候，只会汇报该 GPU 对应的 Extended Resource 的数量。当然，kubelet 本身，会将这个 GPU 的 ID 列表保存在自己的内存里，并通过 ListAndWatch API 定时更新。

而当一个 Pod 想要使用一个 GPU 的时候，它只需要像我在本文一开始给出的例子一样，在 Pod 的 limits 字段声明`nvidia.com/gpu: 1`。那么接下来，Kubernetes 的调度器就会从它的缓存里，寻找 GPU 数量满足条件的 Node，然后将缓存里的 GPU 数量减 1，完成 Pod 与 Node 的绑定。

这个调度成功后的 Pod 信息，自然就会被对应的 kubelet 拿来进行容器操作。而当 kubelet 发现这个 Pod 的容器请求一个 GPU 的时候，kubelet 就会从自己持有的 GPU 列表里，为这个容器分配一个 GPU。此时，kubelet 就会向本机的 Device Plugin 发起一个 Allocate() 请求。这个请求携带的参数，正是即将分配给该容器的设备 ID 列表。

当 Device Plugin 收到 Allocate 请求之后，它就会根据 kubelet 传递过来的设备 ID，从 Device Plugin 里找到这些设备对应的设备路径和驱动目录。当然，这些信息，正是 Device Plugin 周期性的从本机查询到的。比如，在 NVIDIA Device Plugin 的实现里，它会定期访问 nvidia-docker 插件，从而获取到本机的 GPU 信息。

而被分配 GPU 对应的设备路径和驱动目录信息被返回给 kubelet 之后，kubelet 就完成了为一个容器分配 GPU 的操作。接下来，kubelet 会把这些信息追加在创建该容器所对应的 CRI 请求当中。这样，当这个 CRI 请求发给 Docker 之后，Docker 为你创建出来的容器里，就会出现这个 GPU 设备，并把它所需要的驱动目录挂载进去。

至此，Kubernetes 为一个 Pod 分配一个 GPU 的流程就完成了。

对于其他类型硬件来说，要想在 Kubernetes 所管理的容器里使用这些硬件的话，也需要遵循上述 Device Plugin 的流程来实现如下所示的 Allocate 和 ListAndWatch API：

![[Pasted image 20250527220351.png]]
这里最大的问题在于，GPU 等硬件设备的调度工作，实际上是由 kubelet 完成的。即，kubelet 会负责从它所持有的硬件设备列表中，为容器挑选一个硬件设备，然后调用 Device Plugin 的 Allocate API 来完成这个分配操作。可以看到，在整条链路中，调度器扮演的角色，仅仅是为 Pod 寻找到可用的、支持这种硬件设备的节点而已。

这就使得，Kubernetes 里对硬件设备的管理，只能处理“设备个数”这唯一一种情况。一旦你的设备是异构的、不能简单地用“数目”去描述具体使用需求的时候，比如，“我的 Pod 想要运行在计算能力最强的那个 GPU 上”，Device Plugin 就完全不能处理了。

对于硬件资源的管控是很粗粒度的。


# 原理
Device Plugin 的工作原理其实不复杂，可以分为 `插件注册` 和 `kubelet 调用插件`两部分。

- 插件注册：DevicePlugin 启动时会想节点上的 Kubelet 发起注册，这样 Kubelet就可以感知到该插件的存在了
- kubelet 调用插件：注册完成后，当有 Pod 申请对于资源时，kubelet 就会调用该插件 API 实现具体功能

如 k8s 官网上的图所示：
![[Pasted image 20250527220800.png]]

## kubelet 部分
为了提供该功能，Kubelet 新增了一个 `Registration` gRPC service:
```
service Registration {
	rpc Register(RegisterRequest) returns (Empty) {}
}
```
device plugin 可以调用该接口向 Kubelet 进行注册，注册接口需要提供三个参数：

- **device plugin 对应的 unix socket 名字**：后续 kubelet 根据名称找到对应的 unix socket，并向插件发起调用
    
- **device plugin 调 API version**：用于区分不同版本的插件
    
- **device plugin 提供的 ResourceName**：遇到不能处理的资源申请时(CPU和Memory之外的资源)，Kubelet 就会根据申请的资源名称来匹配对应的插件
    - ResourceName 需要按照`vendor-domain/resourcetype` 格式，例如`nvidia.com/gpu`。

## 工作流程
一般所有的 Device Plugin 实现最终都会以 Pod 形式运行在 k8s 集群中，又因为需要管理所有节点，因此都会以 DaemonSet 方式部署。

device plugin 启动之后第一步就是向 Kubelet 注册，让 Kubelet 知道有一个新的设备接入了。

为了能够调用 Kubelet 的 Register 接口，Device Plugin Pod 会将宿主机上的 kubelet.sock 文件(unix socket)挂载到容器中，通过 kubelet.sock 文件发起调用以实现注册。

集群部署后，Kubelet 就会启动，
- 1）Kubelet 启动 Registration gRPC 服务（kubelet.sock），提供 Register 接口
    
- 2）device-plugin 启动后，通过 kubelet.sock 调用 Register 接口，向 Kubelet 进行注册，注册信息包括 device plugin 的 unix socket，API Version，ResourceName
    
- 3）注册成功后，Kubelet 通过 device-plugin 的 unix socket 向 device plugin 调用 ListAndWatch， 获取当前节点上的资源
    
- 4）Kubelet 向 api-server 更新节点状态来记录上一步中发现的资源    
    - 此时 `kubelet get node -oyaml` 就能查看到 Node 对象的 Capacity 中多了对应的资源
    
5）用户创建 Pod 并申请该资源，调度完成后，对应节点上的 kubelet 调用 device plugin 的 Allocate 接口进行资源分配
![[Pasted image 20250527221305.png]]

### GPU 是如何分配给 Pod 的
NVIDIA 提供了 nvidia-container-toolkit 来处理如何将 GPU 分配给容器的问题。

核心组件有以下三个：

- nvidia-container-runtime
    
- nvidia-container-runtime-hook
    
- nvidia-container-cli

首先需要将 docker/containerd 的 runtime 设置为`nvidia-container-runtime`，此后整个调用链就变成这样了：
![[Pasted image 20250528115902.png]]
#### nvidia-container-runtime
nvidia-container-runtime 的作用就是负责在容器启动之前，将 nvidia-container-runtime-hook 注入到 prestart hook。

小知识：docker/containerd 都是高级 Runtime，runC 则是低级 Runtime。不同层级 Runtime 通过 OCI Spec 进行交互。

也就是说 docker 调用 runC 创建容器时，会把 docker 收到的信息解析，组装成 OCI Spec，然后在往下传递。

**而 nvidia-container-runtime 的作用就是修改容器 Spec，往里面添加一个 prestart hook，这个 hook 就是 nvidia-container-runtime-hook** 。

这样 runC 根据 Spec 启动容器时就会执行该 hook，即执行 nvidia-container-runtime-hook。

也就是说 nvidia-container-runtime 其实没有任何逻辑，真正的逻辑都在 nvidia-container-runtime-hook 中。

#### nvidia-container-runtime-hook
nvidia-container-runtime-hook 包含了给容器分配 GPU 的核心逻辑，主要分为两部分：
- 1）从容器 Spec 的 mounts 和 env 中解析 GPU 信息
    - mounts 对应前面 device plugin 中设置的 Mount 和 Device，env 则对应 Env
- 2）调用 `nvidia-container-cli configure` 命令，保证容器内可以使用被指定的 GPU 以及对应能力

也就是说`nvidia-container-runtime-hook` 最终还是调用 `nvidia-container-cli` 来实现的给容器分配 GPU 能力的。

#### nvidia-container-cli
nvidia-container-cli 是一个命令行工具，用于配置 Linux 容器对 GPU 硬件的使用。

提供了三个命令

- list: 打印 nvidia 驱动库及路径
- info: 打印所有Nvidia GPU设备
- configure： 进入给定进程的命名空间，执行必要操作保证容器内可以使用被指定的 GPU 以及对应能力（指定 NVIDIA 驱动库）

一般主要使用 configure 命令，它将 NVIDIA GPU Driver、CUDA Driver 等 驱动库的 so 文件 和 GPU 设备信息， 通过文件挂载的方式映射到容器中。

# 小结
整个流程如下：

- 1）device plugin 上报节点上的 GPU 信息
- 2）用户创建 Pod，在 resources.rquest 中申请 GPU，Scheduler 根据各节点 GPU 资源情况，将 Pod 调度到一个有足够 GPU 的节点
- 3）DevicePlugin 根据 Pod 中申请的 GPU 资源，为容器添加 Env 和 Devices 配置
    - 例如添加环境变量：`NVIDIA_VISIBLE_DEVICES=GPU-03f69c50-207a-2038-9b45-23cac89cb67d`

- 4）docker / containerd 启动容器
    - 由于配置了 nvidia-container-runtime,因此会使用 nvidia-container-runtime 来创建容器
    - nvidia-container-runtime 额外做了一件事：将 `nvidia-container-runtime-hook` 作为 prestart hook 添加到容器 spec 中，然后就将容器 spec 信息往后传给 runC 了。
    - runC 创建容器前会调用 prestart hook，其中就包括了上一步添加的 nvidia-container-runtime-hook，该 hook 主要做两件事：
        - 从容器 Spec 的 mounts 或者 env 中解析 GPU 信息
        - 调用 `nvidia-container-cli configure` 命令，将 NVIDIA 的 GPU Driver、CUDA Driver 等库文件挂载进容器，保证容器内可以使用被指定的 GPU以及对应能力

以上就是在 k8s 中使用 NVIDIA GPU 的流程，简单来说就是：

- 1）device plugin 中根据 pod 申请的 GPU 资源分配 GPU，并以 ENV 环境变量方式添加到容器上。
- 2）nvidia-container-toolkit 则根据该 Env 拿到要分配给该容器的 GPU 最终把相关文件挂载到容器里

当然并不是只有这一种实现方法，比如天数的 [ix-device-plugin](https://github.com/Deep-Spark/ix-device-plugin/blob/master/pkg/dpm/plugin.go#L144-L156) 实现中就没有提供自己的 container-toolkit，只在 device plugin 中通过 Device 指定要挂载哪些设备,这样容器启动时也会把这些设备挂载到容器中：






# allocate 里的 req 是如何生成
下面是你提供的 `allocateContainerResources` 函数的**执行流程分析（控制流 + 状态流）**，帮助你更清晰地理解它的行为：

---

### 🔧 函数签名与作用

```go
func (m *ManagerImpl) allocateContainerResources(pod *v1.Pod, container *v1.Container, devicesToReuse map[string]sets.Set[string]) error
```

> **功能：**  
> 为某个 `Pod` 中的某个 `Container` 分配其所需的设备插件资源，调用对应 device plugin 的 `Allocate()` 方法，并更新内部设备分配状态。

---

## 🧭 执行流程图（文字形式）

### 1️⃣ 初始化变量

- `podUID` 和 `contName` 记录 Pod 和容器的标识符；
    
- `allocatedDevicesUpdated` 表示是否更新过 `allocatedDevices`；
    
- `needsUpdateCheckpoint` 表示是否需要写入 checkpoint。
    

---

### 2️⃣ 遍历 `container.Resources.Limits`

- 对于每一个资源 `k`：
    
    - 如果 `k` 不是 device plugin 管理的扩展资源（通过 `m.isDevicePluginResource` 判断），跳过；
        
    - 如果是第一次遇到 device plugin 资源，调用 `UpdateAllocatedDevices()` 清理旧资源；
        
    - 然后尝试通过 `devicesToAllocate()` 分配资源设备集合。
        

---

### 3️⃣ 分配资源设备集合

```go
allocDevices, err := m.devicesToAllocate(...)
```

- 如果分配失败，返回错误；
    
- 如果分配为空（长度为 0），继续下一个资源；
    
- 否则说明该资源成功分配了一些设备。
    

---

### 4️⃣ 获取对应 device plugin 的 gRPC 端点

```go
eI, ok := m.endpoints[resource]
```

- 如果找不到对应 endpoint，刷新 `allocatedDevices` 并返回错误；
    
- 否则开始 RPC 分配资源：
    

---

### 5️⃣ 调用 device plugin 的 `Allocate()` 方法

```go
resp, err := eI.e.allocate(devs)
```

- 记录 RPC 开始时间；
    
- 失败时恢复 `allocatedDevices` 为 checkpoint 前的状态，并返回错误；
    
- 如果响应中 `ContainerResponses` 为空，也视为错误。
    

---

### 6️⃣ 更新设备 NUMA 拓扑信息

- 遍历 `allocDevices`，根据设备的拓扑信息分类到 `allocDevicesWithNUMA`；
    
- 如果设备没有拓扑信息，则归入 `nodeWithoutTopology`。
    

---

### 7️⃣ 更新 internal podDevices 记录

```go
m.podDevices.insert(...)
```

- 插入分配的设备信息 + AllocateResponse；
    
- 注意：只插入第一个 `ContainerResponses[0]`（假设每次只为一个容器分配）。
    

---

### 8️⃣ 如果有分配成功，写入 checkpoint

```go
if needsUpdateCheckpoint {
    return m.writeCheckpoint()
}
```

---

## ✅ 最终返回值

- 成功则返回 `nil`；
    
- 中途出错则返回相应错误信息（如设备不足、endpoint 不存在、gRPC 失败等）。
    

---

## 📊 简化版执行流程图（伪代码）

```
For each resource in container.Limits:
    If not device-plugin-resource: continue

    If first device plugin resource:
        UpdateAllocatedDevices()

    Try to allocate devices (devicesToAllocate)
    If error: return error
    If no devices: continue

    Lookup device plugin endpoint
    If not found: refresh allocatedDevices, return error

    Call Allocate RPC
    If error: refresh allocatedDevices, return error

    Update NUMA info -> allocDevicesWithNUMA
    Update podDevices with device and response

If any resource allocated: write checkpoint
Return success
```

---

## 📌 总结关键点

|阶段|作用|
|---|---|
|**检测资源类型**|只对 device plugin 资源处理|
|**设备分配**|调用 `devicesToAllocate()`，拿到可用设备|
|**与插件交互**|通过 gRPC `Allocate()` 请求|
|**容错机制**|如果失败会恢复 `allocatedDevices`|
|**状态更新**|更新内部 `podDevices` 映射关系|
|**持久化**|如分配成功，则更新 checkpoint|


好的！我帮你详细分析这个 `devicesToAllocate` 函数的执行流程和关键逻辑。这个函数的主要目标是根据容器资源请求，确定需要分配哪些设备资源（字符串 ID），并返回要向设备插件发起 Allocate RPC 调用的设备列表（或者返回空表示不需要 Allocate 调用）。

---

# 函数功能总结

- 输入：
    
    - podUID、容器名、资源名（例如 GPU）
        
    - 需要分配的设备数 `required`
        
    - 可以重用的设备集合 `reusableDevices`
        
- 输出：
    
    - 需要新分配的设备 ID 集合（`sets.Set[string]`），或者
        
    - 如果不需要分配返回 nil，或者
        
    - 出错返回错误
        

---

# 执行流程详细解析

---

### 1. 加锁，防止并发冲突

```go
m.mutex.Lock()
defer m.mutex.Unlock()
```

所有访问共享状态的操作都在锁内，保证数据一致性。

---

### 2. 计算还需分配的设备数量

- 先取该 pod/container 当前已分配设备：
    

```go
devices := m.podDevices.containerDevices(podUID, contName, resource)
```

- 如果已有设备：
    
    - 减少需要分配的数量 `needed = required - 已分配设备数量`
        
    - 如果 `needed != 0`，说明请求的设备数发生了变化，直接报错。
        

---

### 3. 处理特殊场景：Kubelet 重启和节点重启

- 如果所有资源源（sourcesReady）未准备好，且容器已经是运行状态，则直接返回空，认为设备已经分配，无需操作（kubelet重启场景）。
    

---

### 4. 普通流程：正常需要分配设备

- 检查设备插件是否注册：
    

```go
healthyDevices, hasRegistered := m.healthyDevices[resource]
if !hasRegistered {
    return nil, fmt.Errorf("cannot allocate unregistered device %s", resource)
}
```

- 确认有健康设备：
    

```go
if healthyDevices.Len() == 0 {
    return nil, fmt.Errorf("no healthy devices present; cannot allocate unhealthy devices %s", resource)
}
```

- 检查之前分配的设备是否依然健康：
    

```go
if !healthyDevices.IsSuperset(devices) {
    return nil, fmt.Errorf("previously allocated devices are no longer healthy; cannot allocate unhealthy devices %s", resource)
}
```

- 如果不需要新设备（`needed == 0`），返回空。
    

---

### 5. 定义一个闭包 `allocateRemainingFrom` 用于从给定设备集合中分配设备

- 遍历设备集合中还没分配的设备
    
- 标记已分配，更新内部状态和返回集合
    
- 减少 `needed` 数量
    
- `needed` 减到0时返回 `true` 代表分配完成
    

---

### 6. 尝试先从 `reusableDevices` 中分配设备

- 这里优先复用已有设备
    
- 如果分配完成，直接返回
    

---

### 7. 计算可用设备：

- 当前已经被分配设备：
    

```go
devicesInUse := m.allocatedDevices[resource]
```

- 计算健康且可用设备：
    

```go
available := m.healthyDevices[resource].Difference(devicesInUse)
```

- 如果可用设备数不足，报错。
    

---

### 8. 基于 NUMA 拓扑对可用设备做亲和性划分：

```go
aligned, unaligned, noAffinity := m.filterByAffinity(podUID, contName, resource, available)
```

- 设备根据亲和性被分为：
    
    - `aligned`：符合 NUMA 拓扑亲和性的设备
        
    - `unaligned`：不符合亲和性的设备
        
    - `noAffinity`：无亲和性信息设备
        

---

### 9. 尝试从 `aligned` 集合里分配所有剩余设备（优先分配满足亲和性的）

- 如果需要的数量少于 `aligned` 集合大小：
    
    - 调用插件接口询问优选设备（`callGetPreferredAllocationIfAvailable`）
        
    - 尝试分配优选设备
        
    - 如果优选设备不够，则从 `aligned` 设备分配
        
- 如果需要的设备多于 `aligned` 集合大小：
    
    - 先分配所有 `aligned` 设备
        
    - 再调用插件获取优选设备，尝试分配剩余设备
        
    - 如果插件没返回优选设备，尝试分配 `unaligned` 和 `noAffinity` 设备
        

---

### 10. 如果仍然无法满足分配需求，返回错误

---

# 总结

- 先检查已分配设备是否满足请求
    
- 特殊重启场景下不重复分配
    
- 检查插件注册、设备健康状况
    
- 先尝试复用设备
    
- 从健康可用设备分配，优先满足 NUMA 亲和性
    
- 调用设备插件获取优选分配方案
    
- 设备分配不足时返回错误
    
