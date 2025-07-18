Kubelet 在 Kubernetes 集群中扮演着至关重要的角色，它负责节点上的 Pod 生命周期管理。为了与 Kubernetes API Server 进行安全通信，Kubelet 需要客户端证书。此外，如果启用了 Kubelet 的 HTTPS endpoint，它还需要一个服务证书来供 API Server 或其他组件访问。证书轮转（rotation）是保持集群安全性和可用性的关键，因为它能防止证书过期导致的停机。

Kubelet 启动证书轮转的详细过程主要涉及到 **客户端证书轮转** 和 **服务证书轮转** 两种类型。Kubernetes 提供了自动化机制来处理这些轮转。

---

### Kubelet 客户端证书轮转（Client Certificate Rotation）

Kubelet 客户端证书用于 Kubelet 向 API Server 进行身份验证和授权。默认情况下，这个功能是开启的。

**开启客户端证书轮转的配置：**

- 在 Kubelet 的配置文件（通常是 `/var/lib/kubelet/config.yaml` 或通过 `--config` 参数指定的文件）中设置 `rotateCertificates: true`。
    
- 或者在 Kubelet 启动参数中添加 `--rotate-certificates=true`。
    
- 需要确保 `RotateKubeletClientCertificate` feature gate 被启用 (Kubernetes v1.7 之后默认启用)。
    

**详细轮转过程：**

1. **初始引导（TLS Bootstrap）：**
    
    - 当 Kubelet 首次启动时，它通常没有有效的客户端证书。
        
    - 它会使用一个临时的引导令牌（bootstrap token）或预先配置的 kubeconfig 文件来连接 API Server。
        
    - Kubelet 向 API Server 发送一个 **证书签名请求 (CSR)**。这个 CSR 包含 Kubelet 的身份信息，例如 `system:node:<node-name>` 的 Common Name (CN) 和 `system:nodes` 的组织 (O)。
        
    - API Server 接收到 CSR 后，如果配置了自动批准（例如通过 `kube-controller-manager` 的 `csr-approver`），它会自动批准该 CSR。
        
    - 批准后，`kube-controller-manager` 作为证书签名机构 (CA) 会使用集群 CA 对 CSR 进行签名，并生成 Kubelet 的客户端证书。
        
    - Kubelet 从 API Server 获取到签名的证书后，会将其保存到本地文件系统（通常在 `/var/lib/kubelet/pki/` 目录下）。
        
2. **证书过期检测与续订：**
    
    - Kubelet 会持续监控其当前客户端证书的有效期。
        
    - 当证书即将过期时（通常是证书有效期的大约 70% 处），Kubelet 会自动触发一个新的证书轮转过程。
        
    - Kubelet 会生成一个新的私钥，并使用新私钥生成一个新的 CSR。
        
    - 它再次将这个新的 CSR 发送给 API Server。
        
3. **API Server 批准与签名：**
    
    - API Server 接收到新的 CSR 后，再次进行验证和批准。由于 Kubelet 已经通过现有有效证书进行了身份验证，这个过程通常是无缝的。
        
    - `kube-controller-manager` 再次使用集群 CA 对新的 CSR 进行签名，并生成新的客户端证书。
        
4. **Kubelet 更新证书：**
    
    - Kubelet 从 API Server 获取到新的签名的客户端证书。
        
    - Kubelet 会将新证书和私钥保存到新的文件中 (例如，`/var/lib/kubelet/pki/kubelet-client-xxxx.pem`)，并将其标记为 "current"。
        
    - Kubelet 会更新其内部配置，使其后续与 API Server 的通信使用新的证书。
        
    - 旧的证书文件会保留一段时间，直到其彻底过期或被清理。
        

**优点：**

- **自动化：** 大大降低了手动管理证书的复杂性。
    
- **高可用性：** 防止因证书过期导致的 Kubelet 与 API Server 通信中断。
    
- **安全性：** 定期更换证书可以降低证书被窃取或滥用的风险。
    

---

### Kubelet 服务证书轮转（Serving Certificate Rotation）

Kubelet 服务证书用于 Kubelet 提供 HTTPS API (例如健康检查、`kubectl exec` 等) 时，供其他组件（如 API Server）验证 Kubelet 身份。这个功能默认不开启，需要显式启用。

**开启服务证书轮转的配置：**

- 在 `kube-controller-manager` 的启动参数中添加 `--feature-gates=RotateKubeletServerCertificate=true`。
    
- Kubelet 也需要启用 `rotateCertificates: true`。
    

**详细轮转过程：**

1. **初始引导：**
    
    - 与客户端证书类似，当 `RotateKubeletServerCertificate` 功能启用时，Kubelet 会在启动时生成一个服务端的 CSR。
        
    - 这个 CSR 包含 Kubelet 的节点名称和 IP 地址作为 Subject Alternative Names (SANs)。
        
    - Kubelet 将此 CSR 发送给 API Server。
        
2. **API Server 批准与签名：**
    
    - `kube-controller-manager` 检测到这个特殊的 CSR (通常带有 `system:node-proxier` 或类似的用户信息)。
        
    - 如果 CSR 有效且符合预期，`kube-controller-manager` 会批准并使用集群 CA 签名该 CSR，生成 Kubelet 的服务证书。
        
3. **Kubelet 获取并使用新证书：**
    
    - Kubelet 从 API Server 获取签名的服务证书。
        
    - Kubelet 将新证书和私钥保存到本地（例如 `/var/lib/kubelet/pki/kubelet-server-current.pem`）。
        
    - Kubelet 会重新加载其 HTTPS 服务器，开始使用新的服务证书。
        
    - 类似客户端证书，Kubelet 会持续监控服务证书的有效期，并在临近过期时重复上述过程。
        

**注意事项：**

- **双证书兼容性：** 在证书轮转期间，为了确保兼容性，API Server 可能需要同时信任旧证书和新证书。
    
- **Kubelet 重启：** Kubelet 客户端证书的轮转通常不需要 Kubelet 重启。然而，服务证书的轮转可能需要在 Kubelet 内部重新加载 TLS 配置，这可能导致短暂的服务中断（取决于 Kubelet 的实现）。
    
- **CA 证书轮转：** Kubelet 证书轮转依赖于集群 CA。如果集群的 CA 证书本身需要轮转，那是一个更复杂的过程，通常需要手动干预或使用特定的集群管理工具（如 `kubeadm`、云提供商的工具）来完成。
    

通过这些自动化的证书轮转机制，Kubernetes 大大简化了集群的证书管理，提高了集群的健壮性和安全性。


好的，我们用中文来详细解释 Kubelet 服务证书与 CSI 驱动之间的关系。

---

### Kubelet 服务证书与 CSI 驱动的关系

这个问题很好地触及了 Kubernetes 如何扩展其存储能力的核心！**Kubelet 服务证书**和 **CSI（Container Storage Interface）驱动**通过在每个节点上发生的**安全通信**紧密相关。

---

#### Kubelet 服务证书与 CSI 驱动：安全连接的关键

**Kubelet 服务证书**对于启用 Kubelet HTTPS 端点的安全通信至关重要。Kubernetes API Server 和其他组件（例如各种 Sidecar 容器或外部工具）会使用这个端点与 Kubelet 进行交互。

CSI 驱动程序旨在通过允许外部存储系统无缝集成来扩展 Kubernetes 的存储能力。一个 CSI 驱动程序通常由两个主要组件组成：

1. **控制器插件 (Controller Plugin)：** 这通常作为 Deployment 或 StatefulSet 在 Kubernetes 控制平面或工作节点上运行。它处理全局存储操作，如卷的创建和删除。
    
2. **节点插件 (Node Plugin)：** 这作为 DaemonSet 在 _每个_ 可能使用存储的节点上运行。它负责节点特定的操作，如卷的挂载、卸载和连接。
    

**这就是 Kubelet 服务证书与 CSI 驱动程序关联的地方：**

- **Kubelet 作为节点级存储的编排者：** 当一个 Pod 需要使用 CSI 驱动程序提供的卷时，**该节点上的 Kubelet** 负责向 CSI 驱动程序的节点插件发出必要的调用。
    
- **与 CSI 节点插件的安全通信：** Kubelet 主要通过 **gRPC**（一种高性能的 RPC 框架）通过 **UNIX 域套接字**与 CSI 节点插件进行通信。这些套接字在 Kubelet 和 CSI 驱动程序的节点插件容器之间共享，通常作为 `HostPath` 卷挂载（例如，`/var/lib/kubelet/plugins/csi.sock`）。
    
- **服务证书的作用：** 尽管 Kubelet 和 CSI 节点插件之间的主要通信通道是通过本地 UNIX 域套接字（它本身提供了内在的安全级别，因为它仅限于本地文件系统），但 **Kubelet 的服务证书**对于 **Kubernetes API Server（或其他控制平面组件）**如何安全地与给定节点上的 Kubelet 进行通信至关重要。
    
    - **API Server 到 Kubelet：** 当 API Server 向 Kubelet 发出请求时（例如，查询 Pod 状态、在 Pod 内部执行命令或指示 Kubelet 执行卷操作），它需要信任 Kubelet。**Kubelet 的服务证书**允许 API Server 验证 Kubelet 的身份并确保通信通道是加密的。
        
    - **CSI 控制器到 Kubelet（间接）：** 虽然 CSI 控制器插件直接与 Kubernetes API Server 交互以管理 PersistentVolume 和 PersistentVolumeClaim，但正是 API Server 随后指示 Kubelet 在节点上执行实际的卷操作。为了使这些指令能够安全地传递并由 Kubelet 处理，Kubelet 的服务证书对于 API Server 到 Kubelet 的通信至关重要。
        
    - **健康检查及其他集成：** 任何查询 Kubelet 的 HTTPS 端点以获取节点或 Pod 状态的监控或管理工具都将依赖 Kubelet 的服务证书来建立安全、受信任的连接。
        

**本质上：**

Kubelet 的服务证书确保了 **API Server（以及其他外部组件）与 Kubelet 之间的通信安全**。Kubelet 反过来充当中间人，通过 UNIX 域套接字在本地与 **CSI 节点插件**安全通信，以执行节点上的实际存储操作。

因此，尽管 CSI 驱动程序不 _直接_ 使用 Kubelet 的服务证书与其本地 API（使用 Unix 套接字）进行内部通信，但 Kubernetes 存储生态系统的整体安全运行依赖于 Kubelet 拥有一个有效且受信任的服务证书，以便与 Kubernetes 控制平面的其余部分进行交互。没有它，API Server 将无法安全地指示 Kubelet 管理 CSI 卷。

希望这能清晰地解释它们之间的关系！


Kubelet 在节点上启动后，会立即尝试向 Kubernetes API Server 注册该节点。这个过程通常在 Kubelet 启动的早期阶段发生。

---

### Kubelet 注册节点的时间点及过程

1. **Kubelet 启动时：**
    
    - 当 **Kubelet 进程**在节点上启动时（例如，在操作系统启动后或者通过 Systemd 服务启动），它的首要任务之一就是将自己注册到 Kubernetes 集群。
        
    - 默认情况下，Kubelet 会将 `--register-node` 参数设置为 `true`，这意味着它会尝试**自注册**。
        
    - 它会通过其 `kubeconfig` 文件（该文件包含了连接 API Server 的凭据信息，可能是引导令牌或已签名的证书）连接到 API Server。
        
2. **发送注册请求：**
    
    - Kubelet 向 API Server 发送一个请求，以创建一个新的 **Node 对象**。这个请求包含了节点的基本信息，例如：
        
        - **节点名称 (Node Name)：** 通常是节点的主机名，或者由云提供商（如果配置了云提供商）提供。
            
        - **容量信息 (Capacity)：** 包括 CPU、内存、Pod 数量等资源信息。
            
        - **操作系统信息 (OS Information)：** 操作系统类型、版本等。
            
        - **IP 地址 (IP Addresses)：** 节点的 IP 地址。
            
        - **标签 (Labels)：** 如果 Kubelet 配置了 `--node-labels`，也会包含这些标签。
            
        - **Taints (污点)：** 如果 Kubelet 配置了初始污点，也会包含。
            
3. **API Server 接收与验证：**
    
    - API Server 接收到 Kubelet 的注册请求。
        
    - 如果配置了自动批准（例如，通过 `kube-controller-manager` 的 `csr-approver` 或 `NodeRestriction` Admission Controller），并且请求有效，API Server 会创建或更新对应的 Node 对象。
        
4. **Kubelet 持续报告状态：**
    
    - 一旦节点成功注册，Kubelet 不会停止。它会根据配置的 `--node-status-update-frequency`（默认为 10 秒）定期向 API Server 报告节点的**最新状态**，包括：
        
        - **健康状况 (Conditions)：** 例如 `Ready`（是否健康并准备好接受 Pods）、`MemoryPressure`（内存压力）、`DiskPressure`（磁盘压力）等。
            
        - **已分配资源 (Allocatable)：** 节点上可供 Pods 使用的实际资源。
            
        - **运行中的 Pods 信息。**
            
    - 此外，Kubelet 还会维护一个名为 **Lease 对象**（位于 `kube-node-lease` 命名空间下），并定期（默认为 10 秒）续约此租约，作为节点活跃的“心跳”信号。如果租约长时间未更新，控制平面会认为节点不健康。
        

---

### 总结

Kubelet 注册节点发生在它**启动并成功与 API Server 建立连接之后**。这是一个**自动且持续**的过程：首次启动时创建 Node 对象，之后则不断更新 Node 对象的状态和租约，以确保控制平面始终了解节点的最新情况。这使得调度器能够准确地将 Pods 调度到合适的节点上，并帮助集群管理者监控节点的健康状况。


让我们来详细分解 **Kubelet 在节点上启动并将其注册到 API Server 的完整流程**，这是一个 Kubernetes 节点上线至关重要的第一步。

---

### Kubelet 启动与节点注册的完整流程

Kubelet 是运行在每个 Kubernetes 节点上的主要代理程序。它的核心职责之一就是在启动时向 API Server 注册节点，并持续报告节点状态。

#### 1. Kubelet 进程启动

- **启动方式：** Kubelet 通常以一个系统服务（例如通过 Systemd 在 Linux 上）或容器（如果 Kubelet 本身以容器化方式运行）的形式启动。它会读取其配置文件（通常是 `/var/lib/kubelet/config.yaml`）或命令行参数。
    
- **初始配置加载：** Kubelet 会加载其必要的启动参数，包括：
    
    - **API Server 地址：** 告知 Kubelet 如何找到 API Server。
        
    - **认证凭据：** 连接 API Server 所需的证书、密钥或引导令牌（bootstrap token）。
        
    - **节点名称：** 标识自身的唯一名称，通常是主机名。
        
    - **功能开关 (Feature Gates)：** 启用或禁用某些 Kubernetes 特性。
        
    - **插件目录：** CSI 驱动等插件的套接字路径。
        

#### 2. TLS 引导（TLS Bootstrap）- 获取客户端证书（如果需要）

如果 Kubelet 是首次启动，并且没有预先配置的有效客户端证书来与 API Server 进行安全通信，它会进入 TLS 引导流程：

- **使用引导令牌连接：** Kubelet 会使用一个预设的**引导令牌 (bootstrap token)**（通常在 Kubelet 的 `kubeconfig` 文件中配置）来与 API Server 建立一个临时的、非特权的安全连接。
    
- **创建证书签名请求 (CSR)：** Kubelet 会生成一对新的**私钥**和**证书签名请求 (CSR)**。这个 CSR 包含 Kubelet 的身份信息，如 Common Name (CN) 通常是 `system:node:<node-name>`，以及 `system:nodes` 的组织 (O)。
    
- **提交 CSR：** Kubelet 将这个 CSR 提交给 API Server。
    
- **API Server 批准 CSR：**
    
    - `kube-controller-manager` 中的 `csr-approver`（或手动批准）会识别这是一个来自节点的 CSR。
        
    - 如果 CSR 有效且符合预期的节点命名模式，`kube-controller-manager` 会**自动批准**该 CSR。
        
    - `kube-controller-manager` 会使用集群的**根 CA** 对 CSR 进行签名，生成 Kubelet 的客户端证书。
        
- **Kubelet 获取并保存证书：** Kubelet 从 API Server 获取到签名的客户端证书和集群 CA 证书。它会将这些证书和私钥保存到本地磁盘（通常在 `/var/lib/kubelet/pki/` 目录下），并更新其 `kubeconfig` 文件，以便后续使用这些证书进行身份验证。
    
- **启用证书轮转：** 如果 `rotateCertificates` (客户端证书轮转) 功能开启，Kubelet 会设置一个内部机制来监控此证书的有效期，并在临近过期时自动发起新的 CSR 进行续签。
    

#### 3. 服务证书引导（如果 `RotateKubeletServerCertificate` 启用）

如果 Kubelet 的服务证书轮转功能 (`RotateKubeletServerCertificate` feature gate 在 `kube-controller-manager` 和 Kubelet 端都已启用)，Kubelet 还会为自身的 HTTPS 服务端点执行类似 TLS 引导的流程：

- **生成服务端 CSR：** Kubelet 生成新的私钥和 CSR，其中包含节点的 IP 地址和主机名作为 **Subject Alternative Names (SANs)**。
    
- **提交并获取：** 将 CSR 提交给 API Server，由 `kube-controller-manager` 批准并签名，然后 Kubelet 下载签名的服务证书。
    
- **加载证书：** Kubelet 加载这个服务证书到其 HTTPS 服务器，以便 API Server 或其他组件能够通过 HTTPS 安全地连接到 Kubelet。
    

#### 4. 节点注册到 API Server

一旦 Kubelet 拥有了与 API Server 安全通信的有效凭据（客户端证书），它就可以开始注册节点了：

- **发送 Node 对象创建/更新请求：** Kubelet 向 API Server 发送一个请求，以创建或更新代表该节点的 **Node 对象**。这个请求包含了节点的所有必要信息：
    
    - **元数据：** 节点名称、UID 等。
        
    - **资源容量：** CPU、内存、Pod 数量等。
        
    - **网络信息：** 节点 IP 地址、Pod CIDR 等。
        
    - **操作系统信息：** OS 类型、版本、内核版本等。
        
    - **Kubelet 版本信息：** Kubelet 自身的版本。
        
    - **Node Labels 和 Taints：** 如果 Kubelet 配置了 `--node-labels` 和 `--register-with-taints`，这些信息也会一并发送。
        
    - **Kubelet Configuration：** Kubelet 配置的哈希值等，用于表示节点配置。
        
- **API Server 处理注册请求：**
    
    - API Server 接收到请求后，会验证 Kubelet 的身份（通过其客户端证书）。
        
    - 如果验证通过，并且请求有效，API Server 会在集群中**创建或更新**对应的 Node 对象。
        
    - `NodeRestriction` Admission Controller 可能会参与验证，确保 Kubelet 只能修改自己的 Node 对象。
        

#### 5. Kubelet 持续同步节点状态与心跳

节点注册成功后，Kubelet 的任务远未结束：

- **定期更新 Node 状态：** Kubelet 会根据配置的 `--node-status-update-frequency`（默认为 10 秒），定期向 API Server 发送节点状态更新。这些状态包括：
    
    - **条件 (Conditions)：** 例如 `Ready`（表示节点是否健康且可以调度 Pods）、`MemoryPressure`、`DiskPressure`、`PIDPressure` 等。
        
    - **已分配资源 (Allocatable)：** 节点上实际可供 Pods 使用的资源量。
        
    - **正在运行的 Pod 列表：** Kubelet 会报告其上正在运行的 Pods 的列表和状态。
        
    - **事件 (Events)：** 报告节点相关的事件。
        
- **Lease 对象心跳：** Kubelet 还会定期（默认为 10 秒）更新其对应的 **Lease 对象**（位于 `kube-node-lease` 命名空间下）。这个 Lease 对象作为节点的“心跳”，向控制平面表明节点仍然活跃。如果 Lease 对象在一定时间内没有被更新，API Server 会将 Node 对象标记为不健康 (`NotReady`)，并触发相应的调度或驱逐行为。
    

---

**总结来说，Kubelet 启动并注册节点的完整流程是一个自举、安全且持续的过程：**

1. **启动加载配置。**
    
2. **（如果需要）通过 TLS 引导获取客户端证书，确保持久安全通信。**
    
3. **（如果启用）通过 TLS 引导获取服务证书，确保持续服务安全。**
    
4. **使用有效凭据向 API Server 发送节点信息，注册或更新 Node 对象。**
    
5. **持续向 API Server 报告节点状态并更新租约，维护节点的活跃和健康信息。**
    

这个流程确保了每个节点都能安全、准确地融入 Kubernetes 集群，为 Pod 的调度和运行做好准备。


在 Kubernetes 中，kubelet 从自启动到成功注册节点（Successfully registered node）是一个关键且复杂的过程，其中 **TLS Bootstrap** 扮演着至关重要的角色。这个过程确保了新加入的节点能够安全地与 Kubernetes 控制平面通信。

---

## 1. Kubelet 自启动

当一个 Kubernetes 工作节点启动时，`kubelet` 服务会随之启动。`kubelet` 是运行在每个节点上的主要 "节点代理"，负责维护节点上的 Pod，并与 API Server 进行通信。

Kubelet 启动时会尝试加载其配置，这通常包括指定 API Server 的地址以及用于认证的凭据。如果 Kubelet 发现它还没有有效的客户端证书和密钥来与 API Server 进行安全的 TLS 通信，它就会进入 TLS Bootstrap 流程。

---

## 2. TLS Bootstrap 流程

TLS Bootstrap 的核心目的是自动化 Kubelet 客户端证书的颁发和管理。在 Kubernetes 1.4 之前，这个过程是手动完成的，非常繁琐且容易出错。TLS Bootstrap 机制大大简化了节点的加入过程。

以下是 TLS Bootstrap 的详细步骤：

### 2.1. 查找 Bootstrap Kubeconfig

- **初始启动检查**: 当 Kubelet 首次启动时，它会检查 `--kubeconfig` 参数指定的文件是否存在。如果该文件不存在或无效，Kubelet 会尝试查找由 `--bootstrap-kubeconfig` 参数指定的**引导 Kubeconfig 文件**。
    
- **引导 Kubeconfig 的作用**: 这个引导 Kubeconfig 文件是一个临时的、权限受限的 Kubeconfig 文件，它包含了一个**引导令牌 (Bootstrap Token)**，而不是完整的 TLS 客户端证书。这个引导令牌是预先由集群管理员创建并分发到新节点的。
    

### 2.2. 使用 Bootstrap Token 进行初步认证

- **连接 API Server**: Kubelet 使用引导 Kubeconfig 中的引导令牌连接到 API Server。由于引导令牌是明文的 bearer token，因此 Kubelet 在这个阶段会接受自签名的 API Server 证书，或者通过预先配置的 CA 证书验证 API Server。
    
- **身份认证**: API Server 接收到 Kubelet 的请求后，会使用 `Bootstrap Authenticator` 来验证这个引导令牌。如果令牌有效，Kubelet 会被认证为一个特定的用户（例如 `system:bootstrap:<token-id>`）并加入到 `system:bootstrappers` 组。
    

### 2.3. 创建 Certificate Signing Request (CSR)

- **生成密钥对**: Kubelet 在本地生成一个新的私钥和公钥对。这个私钥将用于其未来的 TLS 客户端证书。
    
- **提交 CSR**: Kubelet 使用其引导令牌认证的会话，向 API Server 提交一个 **Certificate Signing Request (CSR)**。这个 CSR 包含 Kubelet 生成的公钥以及一些请求的证书属性（例如 `Common Name` 通常是 `system:node:<node-name>`，`Organization` 通常是 `system:nodes`）。
    
- **RBAC 授权**: 为了让 Kubelet 能够提交 CSR，集群中的 RBAC 规则必须允许 `system:bootstrappers` 组的用户创建 `certificatesigningrequests` 资源。通常，这通过一个名为 `system:node-bootstrapper` 的 ClusterRole 和相应的 ClusterRoleBinding 来实现。
    

### 2.4. CSR 的审批与签名

- **等待审批**: 提交 CSR 后，Kubelet 会等待这个 CSR 被集群管理员或自动审批控制器审批。
    
- **自动审批 (推荐)**: 在生产环境中，通常会配置一个自动审批控制器（例如 `kube-controller-manager` 中的 `csrapproving` 控制器），它会监控新的 CSRs。如果 CSR 来自 `system:bootstrappers` 组，并且满足特定的条件（例如请求的 Common Name 符合 `system:node:<node-name>` 模式），该控制器会自动将 CSR 标记为 `Approved`。
    
- **手动审批 (不推荐)**: 在某些情况下，如果自动审批没有配置或出现问题，管理员可以手动审批 CSR，例如使用 `kubectl certificate approve <csr-name>` 命令。
    
- **证书签名**: 一旦 CSR 被审批，`kube-controller-manager` 中的 `CSR Signer` 控制器会使用集群的 CA 证书和密钥来签名 Kubelet 的客户端证书，并将其作为 CSR 对象的一部分存储在 API Server 中。
    

### 2.5. 下载并保存客户端证书

- **获取签名证书**: Kubelet 会定期轮询 API Server，检查其提交的 CSR 状态。当它发现 CSR 已经被签名后，会下载签名的客户端证书。
    
- **生成 Kubeconfig**: Kubelet 将下载的客户端证书和之前生成的私钥组合起来，创建一个完整的 Kubeconfig 文件。这个 Kubeconfig 文件将永久地存储在 `--kubeconfig` 参数指定的路径（通常是 `/var/lib/kubelet/kubeconfig`）中，替换掉临时的引导 Kubeconfig。
    
- **清除引导令牌**: 一旦 Kubelet 获得了有效的客户端证书，它就不再需要引导令牌了。
    

---

## 3. 节点成功注册

在 Kubelet 成功获取了客户端证书后，它就可以使用这个证书进行认证，并开始与 API Server 进行常态化、安全的 TLS 通信了。

- **健康检查与状态报告**: Kubelet 会定期向 API Server 报告其节点的状态、资源使用情况、可分配资源、以及运行的 Pods 信息。
    
- **节点注册**: Kubelet 通过向 API Server 创建或更新一个 `Node` 对象来完成节点的注册。这个 `Node` 对象包含了节点的元数据（如 hostname、IP 地址、操作系统、架构等）和能力信息。
    
- **"Successfully registered node"**: 当 API Server 成功接收并处理了 Kubelet 报告的节点信息，并且节点对象在集群中被创建或更新时，Kubelet 的日志中就会出现类似 `"Successfully registered node <node-name>"` 的消息。这意味着节点现在已经被 Kubernetes 控制平面所识别，并且可以调度 Pod 到该节点上运行了。
    

---

## 总结流程图

1. **Kubelet 启动**
    
    - 检查 `/var/lib/kubelet/kubeconfig` 是否存在有效证书。
        
    - 如果不存在，读取 `--bootstrap-kubeconfig`。
        
2. **初步认证 (使用 Bootstrap Token)**
    
    - Kubelet 使用 `--bootstrap-kubeconfig` 中的引导令牌向 API Server 发起请求。
        
    - API Server 通过 `Bootstrap Authenticator` 验证令牌，将 Kubelet 认证为 `system:bootstrappers` 组的成员。
        
3. **提交 CSR**
    
    - Kubelet 本地生成新的 TLS 密钥对。
        
    - Kubelet 创建一个 CSR，包含公钥和节点信息，并通过初步认证的会话提交给 API Server。
        
4. **CSR 审批与签名**
    
    - `kube-controller-manager` 中的 `csrapproving` 控制器（如果配置）自动审批 CSR。
        
    - `CSR Signer` 控制器使用集群 CA 签名 CSR，生成 Kubelet 客户端证书。
        
5. **获取并保存证书**
    
    - Kubelet 轮询 API Server 获取签名的客户端证书。
        
    - 将签名证书和私钥保存到 `/var/lib/kubelet/kubeconfig`。
        
6. **安全通信与节点注册**
    
    - Kubelet 使用新的客户端证书与 API Server 建立安全的 TLS 连接。
        
    - Kubelet 向 API Server 汇报节点状态、资源信息，创建或更新 Node 对象。
        
    - **节点成功注册**: API Server 确认节点信息，Kubelet 日志显示 "Successfully registered node"。
        

这个过程确保了新节点加入 Kubernetes 集群时的安全性和自动化，避免了手动管理证书的复杂性，同时也为集群的扩展提供了便利。