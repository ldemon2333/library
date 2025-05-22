- Define the k8s architecture
- Explain the different components of the control plane and worker nodes
- Discuss cluster state management with etcd
- Review the k8s network setup requirements

![[Pasted image 20250507235146.png]]
- One or more control plane nodes
- One or more worker nodes

# Control Plane Node Overview
不惜一切代价保持控制平面运行至关重要。控制平面故障可能导致停机，从而导致客户端服务中断，甚至可能导致业务损失。为了确保控制平面的容错能力，可以将控制平面节点副本添加到集群中，并配置为高可用性 (HA)。虽然只有一个控制平面节点专用于主动管理集群，但控制平面组件在控制平面节点副本之间保持同步。这种配置可以在主动控制平面节点发生故障时增强集群控制平面的弹性。

为了持久化 Kubernetes 集群的状态，所有集群配置数据都保存在一个分布式键值存储中，该存储仅保存集群状态相关数据，不保存客户端工作负载生成的数据。该键值存储可以配置在控制平面节点（堆叠拓扑）上，也可以配置在其专用主机（外部拓扑）上，以便通过将其与其他控制平面代理解耦来降低数据存储丢失的可能性。

在堆叠式键值存储拓扑中，HA 控制平面节点副本也能确保键值存储的弹性。然而，外部键值存储拓扑则并非如此，因为专用键值存储主机必须单独复制以实现 HA，这种配置需要额外的硬件，从而增加运营成本。

A control plane node runs the following essential control plane components and agents:

- API Server
- Scheduler
- Controller Managers
- Key-Value Data Store

In addition, the control plane node runs:

- Container Runtime
- Node Agent - kubelet
- Proxy - kube-proxy
- Optional add-ons for observability, such as dashboards, cluster-level monitoring, and logging

## API Server
所有管理任务均由 kube-apiserver 协调，它是一个运行在控制平面节点上的中央控制平面组件。API 服务器拦截来自用户、管理员、开发者、运维人员和外部代理的 RESTful 调用，然后验证并处理这些调用。在处理过程中，API 服务器从键值存储中读取 Kubernetes 集群的当前状态，调用执行后，Kubernetes 集群的最终状态将保存在键值存储中以实现持久化。API 服务器是唯一与键值存储交互的控制平面组件，它既可以读取 Kubernetes 集群状态信息，也可以保存 Kubernetes 集群状态信息——充当任何其他查询集群状态的控制平面代理的中间接口。

API 服务器高度可配置且可定制。它支持水平扩展，同时也支持添加自定义的辅助 API 服务器。这种配置会将主 API 服务器转换为所有辅助自定义 API 服务器的代理，并根据自定义规则将所有传入的 RESTful 调用路由到这些服务器。


## Control Plane Node Components: Scheduler
kube-scheduler 的作用是将新的工作负载对象（例如封装容器的 Pod）分配给节点（通常是工作节点）。在调度过程中，系统会根据当前 Kubernetes 集群状态和新工作负载对象的需求做出决策。调度器通过 API 服务器从键值存储中获取集群中每个工作节点的资源使用情况数据。调度器还会从 API 服务器接收新工作负载对象的配置数据，这些配置数据是其配置数据的一部分。这些配置数据可能包括用户和运维人员设置的约束，例如将工作调度到带有 disk == ssd 键值对的节点上。调度器还会考虑服务质量 (QoS) 要求、数据本地性、亲和性、反亲和性、污点、容忍度、集群拓扑等。一旦所有集群数据可用，调度算法就会使用谓词筛选节点，以分离出可能的候选节点，然后根据优先级对这些候选节点进行评分，最终选择一个能够满足所有新工作负载托管需求的节点。决策过程的结果被传达回 API 服务器，然后 API 服务器将工作负载部署委托给其他控制平面代理。


## Controller Managers
The **controller managers** are components of the control plane node running controllers or operator processes to regulate the state of the Kubernetes cluster. 

The **kube-controller-manager** runs controllers or operators responsible to act when nodes become unavailable, to ensure container pod counts are as expected, to create endpoints, service accounts, and API access tokens.

The **cloud-controller-manager** runs controllers or operators responsible to interact with the underlying infrastructure of a cloud provider when nodes become unavailable, to manage storage volumes when provided by a cloud service, and to manage load balancing and routing.

## etcd
etcd 是一个强一致性、分布式键值数据存储，用于持久化 Kubernetes 集群的状态。新数据仅通过追加的方式写入数据存储，数据永远不会被替换。过时的数据会被定期压缩（或拆分），以最小化数据存储的大小。

Out of all the control plane components, only the API Server is able to communicate with the etcd data store.

etcd 的 CLI 管理工具 etcdctl 提供了快照保存和恢复功能，这对于单 etcd 实例 Kubernetes 集群尤其有用——这在开发和学习环境中很常见。然而，在阶段和生产环境中，为了确保集群配置数据弹性，以 HA 模式复制数据存储至关重要。

![[Pasted image 20250508003446.png]]

![[Pasted image 20250508003451.png]]

etcd 基于 Raft 共识算法，该算法允许一组机器以一致的组形式工作，即使部分成员发生故障也能继续运行。在任何给定时间，组中的一个节点将成为领导者，其余节点将成为追随者。etcd 能够优雅地处理领导者选举，并能够容忍节点故障，包括领导者节点故障。任何节点都可以被视为领导者。

Keep in mind however, that the leader/followers hierarchy is distinct from the primary/secondary hierarchy, meaning that neither node is favored for the leader role, and neither node outranks other nodes. A leader will remain active until it fails, at which point in time a new leader is elected by the group of healthy followers.
![[Pasted image 20250508003657.png]]




`Leader/Follower` 和 `Primary/Secondary` 都是分布式系统中常见的**主从结构模型**，它们有很多相似之处，但也存在**语义、适用场景和角色职责**方面的差异。

---

## ✅ 1. 定义对比

|概念|Leader/Follower|Primary/Secondary|
|---|---|---|
|含义|一台节点为 Leader，其它为 Follower，**Leader 负责所有写入，Follower 复制**|一台为 Primary，其它为 Secondary，**Primary 通常负责写，Secondary 可用于读或备份**|
|常见场景|分布式一致性系统（如 Raft、Kafka、ZooKeeper）|数据库主从（如 MySQL 主从、MongoDB）|
|选主机制|多为自动选主（通过共识算法如 Raft、Paxos）|有时支持人工切换主，或者简化选主逻辑|
|强一致性|强一致性保障更强（如写入顺序）|一致性不一定严格，允许最终一致或读写分离|

---

## ✅ 2. 语义差异

### Leader / Follower 模型

- 强调的是 **一致性 + 协调者角色**
    
- **Leader 统一处理指令（写操作、调度）**
    
- Follower 被动复制 Leader 的日志/状态，通常不能提供写服务
    
- 强调**一致性和调度控制**
    

📌 常见于：

- Raft、Paxos、ZooKeeper、Kafka
    
- Kubernetes 控制平面（etcd 也是基于 Raft）
    

---

### Primary / Secondary 模型

- 更偏向于 **读写职责分工**
    
- Primary 处理写请求，Secondary 多数只读，有时用于备份或横向扩展读能力
    
- 复制机制可能是异步的 → 一致性可能是“最终一致性”
    

📌 常见于：

- 数据库领域（MySQL、MongoDB、PostgreSQL 逻辑复制）
    
- 存储系统中的数据同步
    

---

## ✅ 3. 对比总结

|对比项|Leader/Follower|Primary/Secondary|
|---|---|---|
|强调内容|一致性与协调|数据读写职责|
|是否自动选主|通常自动（如 Raft）|有些系统允许手动|
|写请求处理|仅 Leader|仅 Primary|
|读请求处理|通常只 Leader（也可 Follower，需同步）|Secondary 可读（读写分离）|
|是否强一致性|是（如 Raft）|不一定（可异步复制）|
|应用场景|分布式一致性协议、日志同步|数据库主从复制、读写分离系统|

---

## ✅ 举例说明

|系统|模型类型|说明|
|---|---|---|
|**Kafka**|Leader / Follower|每个分区一个 Leader，其它为 Follower|
|**etcd**|Leader / Follower|使用 Raft 协议选主|
|**ZooKeeper**|Leader / Follower|多节点协作，Leader 提交变更|
|**MySQL 主从**|Primary / Secondary|Primary 写，Secondary 读|
|**MongoDB 副本集**|Primary / Secondary|Primary 处理写操作，Secondary 可读可提升为主|

# Worker Node Overview
A worker node has the following components:

- Container Runtime
- Node Agent - kubelet
- Proxy - kube-proxy
- Add-ons for DNS, observability components such as dashboards, cluster-level monitoring and logging, and device plugins.

## kubelet
![[Pasted image 20250508113346.png]]
kubelet 是在每个节点、控制平面和工作器上运行的代理，它与控制平面进行通信。它主要从 API Server 接收 Pod 定义，并与节点上的容器运行时交互以运行与 Pod 关联的容器。 It also monitors the health and resources of Pods running containers.

The kubelet connects to container runtimes through a plugin based interface - the [Container Runtime Interface](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/container-runtime-interface.md) (CRI). The CRI consists of protocol buffers, gRPC API, libraries, and additional specifications and tools. In order to connect to interchangeable container runtimes, kubelet uses a **CRI shim**, an application which provides a clear abstraction layer between kubelet and the container runtime.

As shown above, the kubelet acting as grpc client connects to the CRI shim acting as grpc server to perform container and image operations. The CRI implements two services: **ImageService** and **RuntimeService**. The **ImageService** is responsible for all the image-related operations, while the **RuntimeService** is responsible for all the Pod and container-related operations.


## kubelet - CRI shims
最初，kubelet 代理仅支持少数几个容器运行时，首先是 Docker Engine，然后是 rkt，并通过直接集成在 kubelet 源代码中的独特接口模型来实现。然而，尽管这种方法对 Docker 尤其有益，但它并非注定要永远持续下去。随着时间的推移，Kubernetes 开始通过引入 CRI 向标准化的容器运行时集成方法迁移。Kubernetes 采用了一种解耦且灵活的方法，无需重新编译其源代码即可与各种容器运行时集成。任何实现 CRI 的容器运行时都可以被 Kubernetes 用来管理容器。

Shim 是容器运行时接口 (CRI) 的实现，具体来说，是 Kubernetes 支持的每个容器运行时的接口或适配器。下面我们介绍一些 CRI Shim 的示例：

### cri-containerd
cri-containerd allows containers to be directly created and managed with containerd at kubelet's request:
![[Pasted image 20250509152013.png]]
### CRI-O
CRI-O enables the use of any Open Container Initiative (OCI) compatible runtime with Kubernetes, such as runC:
![[Pasted image 20250509152134.png]]

## Proxy - kube-proxy
network agent which runs on each node, control plane and workers, responsible for dynamic updates and maintenance of all networking rules on the node. 

kube-proxy 负责跨应用程序的一组 Pod 后端进行 TCP、UDP 和 SCTP 流转发或随机转发，并通过 Service API 对象实现用户定义的转发规则。

kube-proxy 节点代理与节点的 iptables 协同运行。iptables 是一款为 Linux 操作系统创建的防火墙实用程序，用户可以通过同名的 CLI 实用程序进行管理。iptables 实用程序适用于许多 Linux 发行版，并且已预装在发行版中。

For system hardware resources, such as GPU, FPGA, high-performance NIC, to be advertised by the node to application pods.

# Networking Challenges
- Container-to-Container communication inside Pods
- Pod-to-Pod communication on the same node and across cluster nodes
- Service-to-Pod communication within the same namespace and across cluster namespaces
- External-to-Service communication for clients to access applications in a cluster

容器运行时利用底层主机操作系统的内核虚拟化功能，为其启动的每个容器创建一个隔离的网络空间。在 Linux 上，此隔离的网络空间称为网络命名空间。网络命名空间可以在容器之间共享，也可以与主机操作系统共享。

当 Pod 定义的容器组启动时，容器运行时会初始化一个特殊的基础设施“暂停”容器，其唯一目的是为 Pod 创建网络命名空间。所有通过用户请求创建并在 Pod 内运行的其他容器都将共享“暂停”容器的网络命名空间，以便它们可以通过 localhost 相互通信。

## Pod-to-Pod
在 Kubernetes 集群中，Pod（容器组）以几乎不可预测的方式在节点上进行调度。无论 Pod 位于哪个主机节点，它们都应能够与集群中的所有其他 Pod 进行通信，而无需实现网络地址转换 (NAT)。这是 Kubernetes 中任何网络实现的基本要求。

Kubernetes 网络模型旨在降低复杂性，它将 Pod 视为网络上的虚拟机，每个虚拟机都配备一个网络接口，因此每个 Pod 都拥有一个唯一的 IP 地址。这种模型被称为“每个 Pod 一个 IP”，确保 Pod 之间的通信，就像虚拟机能够在同一网络上相互通信一样。

不过，我们不要忘记容器。它们共享 Pod 的网络命名空间，并且必须像虚拟机上的应用程序一样协调 Pod 内部的端口分配，同时还能够在 Pod 内部的本地主机上相互通信。然而，容器通过使用由 CNI 插件支持的容器网络接口 (CNI) 与整体 Kubernetes 网络模型集成。CNI 是一组规范和库，允许插件配置容器的网络。虽然有一些核心插件，但大多数 CNI 插件都是实现 Kubernetes 网络模型的第三方软件定义网络 (SDN) 解决方案。除了满足网络模型的基本需求外，一些网络解决方案还提供对网络策略的支持。Flannel、Weave、Calico 和 Cilium 只是 Kubernetes 集群可用的 SDN 解决方案中的一小部分。

![[Pasted image 20250509153535.png]]
容器运行时将 IP 分配任务交给 CNI，CNI 会连接到底层已配置的插件（例如 Bridge 或 MACvlan）以获取 IP 地址。相应插件获取 IP 地址后，CNI 会将其转发回请求的容器运行时。

## External-to-Pod
成功部署并运行在 Kubernetes 集群内部 Pod 中的容器化应用程序可能需要外部可访问性。Kubernetes 通过服务 (Service) 实现外部可访问性。服务是对网络路由规则定义的复杂封装，存储在集群节点的 iptables 中，并由 kube-proxy 代理实现。通过借助 kube-proxy 将服务暴露给外部，应用程序可以通过虚拟 IP 地址和专用端口号从集群外部访问。

