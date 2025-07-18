# fields
A Node's status contains the following information:
- [Addresses](https://kubernetes.io/docs/reference/node/node-status/#addresses)
- [Conditions](https://kubernetes.io/docs/reference/node/node-status/#condition)
- [Capacity and Allocatable](https://kubernetes.io/docs/reference/node/node-status/#capacity)
- [Info](https://kubernetes.io/docs/reference/node/node-status/#info)

# Addresses
The usage of these fields varies depending on your cloud provider or bare metal configuration.

- HostName: The hostname as reported by the node's kernel. Can be overridden via the kubelet `--hostname-override` parameter.
- ExternalIP: Typically the IP address of the node that is externally routable (available from outside the cluster).
- InternalIP: Typically the IP address of the node that is routable only within the cluster.


# Conditions
The `conditions` field describes the status of all `Running` nodes. Examples of conditions include:

|Node Condition|Description|
|---|---|
|`Ready`|`True` if the node is healthy and ready to accept pods, `False` if the node is not healthy and is not accepting pods, and `Unknown` if the node controller has not heard from the node in the last `node-monitor-grace-period` (default is 50 seconds)|
|`DiskPressure`|`True` if pressure exists on the disk size—that is, if the disk capacity is low; otherwise `False`|
|`MemoryPressure`|`True` if pressure exists on the node memory—that is, if the node memory is low; otherwise `False`|
|`PIDPressure`|`True` if pressure exists on the processes—that is, if there are too many processes on the node; otherwise `False`|
|`NetworkUnavailable`|`True` if the network for the node is not correctly configured, otherwise `False`|

## Note:
If you use command-line tools to print details of a cordoned Node, the Condition includes `SchedulingDisabled`. `SchedulingDisabled` is not a Condition in the Kubernetes API; instead, cordoned nodes are marked Unschedulable in their spec.

In the Kubernetes API, a node's condition is represented as part of the `.status` of the Node resource. For example, the following JSON structure describes a healthy node:

```json
"conditions": [
  {
    "type": "Ready",
    "status": "True",
    "reason": "KubeletReady",
    "message": "kubelet is posting ready status",
    "lastHeartbeatTime": "2019-06-05T18:38:35Z",
    "lastTransitionTime": "2019-06-05T11:41:27Z"
  }
]
```

When problems occur on nodes, the Kubernetes control plane automatically creates [taints](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) that match the conditions affecting the node. An example of this is when the `status` of the Ready condition remains `Unknown` or `False` for longer than the kube-controller-manager's `NodeMonitorGracePeriod`, which defaults to 50 seconds. This will cause either an `node.kubernetes.io/unreachable` taint, for an `Unknown` status, or a `node.kubernetes.io/not-ready` taint, for a `False` status, to be added to the Node.

These taints affect pending pods as the scheduler takes the Node's taints into consideration when assigning a pod to a Node. Existing pods scheduled to the node may be evicted due to the application of `NoExecute` taints. Pods may also have [tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) that let them schedule to and continue running on a Node even though it has a specific taint.

See [Taint Based Evictions](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/#taint-based-evictions) and [Taint Nodes by Condition](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/#taint-nodes-by-condition) for more details.

# Capacity and Allocatable
Describes the resources available on the node: CPU, memory, and the maximum number of pods that can be scheduled onto the node.

The fields in the capacity block indicate the total amount of resources that a Node has. The allocatable block indicates the amount of resources on a Node that is available to be consumed by normal Pods.

You may read more about capacity and allocatable resources while learning how to [reserve compute resources](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#node-allocatable) on a Node.

# Info
Describes general information about the node, such as kernel version, Kubernetes version (kubelet and kube-proxy version), container runtime details, and which operating system the node uses. The kubelet gathers this information from the node and publishes it into the Kubernetes API.

# Heartbeats
Heartbeats, sent by Kubernetes nodes, help your cluster determine the availability of each node, and to take action when failures are detected.

For nodes there are two forms of heartbeats:
- updates to the `.status` of a Node
- [Lease](https://kubernetes.io/docs/concepts/architecture/leases/) objects within the `kube-node-lease` [namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces). Each Node has an associated Lease object.

Compared to updates to `.status` of a Node, a Lease is a lightweight resource. Using Leases for heartbeats reduces the performance impact of these updates for large clusters.

The kubelet is responsible for creating and updating the `.status` of Nodes, and for updating their related Leases.

- The kubelet updates the node's `.status` either when there is change in status or if there has been no update for a configured interval. The default interval for `.status` updates to Nodes is 5 minutes, which is much longer than the 40 second default timeout for unreachable nodes.
- The kubelet creates and then updates its Lease object every 10 seconds (the default update interval). Lease updates occur independently from updates to the Node's `.status`. If the Lease update fails, the kubelet retries, using exponential backoff that starts at 200 milliseconds and capped at 7 seconds.

---

### 什么是 Lease Object？

在 Kubernetes (K8s) 中，**Lease Object**（租约对象）是一种特殊的非持久性资源，主要用于实现**分布式锁**和**心跳机制**，以确保集群中各个组件的**高可用性**和**一致性**。它在 K8s API 中以 `coordination.k8s.io/v1` API 组的形式存在。

---

### Lease Object 的核心作用

Lease Object 最核心的作用体现在以下两个方面：

1. **Leader 选举 (Leader Election)**
    
    - 这是 Lease 最常见的用途。在 K8s 集群中，许多控制器（例如 Deployment Controller、StatefulSet Controller 等）或外部操作器 (Operator) 都是**单例模式**运行的，即在任何给定时间点，只能有一个实例处于“活动”（Leader）状态来避免冲突和重复操作。
        
    - 这些组件会尝试获取一个特定的 Lease 对象。成功获取并持有 Lease 的实例，就被认为是当前的 Leader。
        
    - **工作原理**:
        
        - Leader 会定期更新 Lease 对象的 `renewTime` 字段，表明它仍然“活着”并持有租约。
            
        - 如果 Leader 崩溃或网络中断，它将无法更新 Lease。
            
        - 在 Lease 的 `leaseDurationSeconds`（租约时长）过期后，其他等待中的实例会尝试获取该 Lease。
            
        - 第一个成功获取到 Lease 的实例就成为了新的 Leader，从而实现了控制器的**高可用性**。
            
2. **节点心跳 (Node Heartbeat)**
    
    - 在 Kubernetes 1.14 版本之后，Lease Object 也被用于实现节点心跳机制（`NodeLease`）。每个 Kubelet（节点上的代理）会定期更新一个与其节点名称对应的 Lease 对象。
        
    - 这个 Lease 的更新行为充当了 Kubelet 向 API Server 报告其**健康状况和活跃性**的心跳。
        
    - **好处**:
        
        - **减轻 API Server 负担**: 节点不再需要频繁地更新整个 Node 对象（其中包含大量信息），只需更新一个小的 Lease 对象即可报告心跳，这大大减少了 API Server 的写操作负载。
            
        - **提高可扩展性**: 允许集群支持更多的节点。
            
        - **更快的节点故障检测**: 如果一个节点的 Lease 长期未更新，API Server 会更快地判断该节点为不健康或失联，从而触发相应的调度或恢复操作。
            

---

### Lease Object 的关键字段

一个典型的 Lease Object 可能包含以下重要字段：

- **`holderIdentity`**: 字符串类型，标识当前持有租约的客户端的身份（例如，一个控制器实例的 Pod 名称和 UID）。
    
- **`leaseDurationSeconds`**: 整数类型，指定租约的有效期（秒）。如果持有者在此期间没有更新租约，则租约过期。
    
- **`acquireTime`**: 时间戳，表示租约被获取的时间。
    
- **`renewTime`**: 时间戳，表示租约最后一次被更新的时间。持有者需要在此字段过期前更新它。
    
- **`leaderElection`**: 布尔类型，如果此 Lease 用于 Leader 选举，则为 `true`。
    

---

### 总结

Lease Object 是 Kubernetes 内部协调和维持集群健康状态的关键机制。它通过提供一种轻量级的分布式锁和心跳通信方式，**有效支持了控制器的 Leader 选举，确保了高可用性，并优化了节点心跳机制，显著提升了集群的可伸缩性和稳定性**。对于 Kubernetes 的稳定运行和故障恢复来说，Lease Object 是一个幕后英雄。


当一个 Kubernetes 节点（通常运行着 **kubelet**）第一次注册到 API Server 时，它的状态会经历一个初始过程，通常最终会达到 **`Ready`** 状态。

---

### 节点注册的初始状态流程

1. **节点对象创建：**
    
    - 当节点上的 **kubelet** 启动时，如果其配置中设置了 `--register-node=true`（这是默认值），kubelet 会尝试向 Kubernetes API Server 注册自己。
        
    - 这个注册过程会在 API Server 中创建一个代表该节点的新 **Node 对象**。
        
    - 最初，这个 Node 对象可能还没有完整的状态信息，或者它可能被标记为 **`Pending`**（尽管这在 `kubectl get nodes` 输出中不常直接看到，更多是内部状态）。
        
2. **Kubelet 心跳与状态更新：**
    
    - 一旦 Node 对象被创建，kubelet 会开始定期向 API Server 发送**心跳 (heartbeat)** 和**状态更新**。
        
    - 它会收集自身的信息，包括：
        
        - **地址 (Addresses):** 节点的 IP 地址（InternalIP, ExternalIP）。
            
        - **容量 (Capacity):** 节点的 CPU、内存、存储等资源总量。
            
        - **信息 (Info):** 操作系统、内核版本、容器运行时信息等。
            
        - **条件 (Conditions):** 这部分是关键。
            
3. **初始的 Condition 状态：**
    
    - 在刚注册时，节点可能会报告一些初始的 **Conditions**，但其中最重要的就是 **`Ready`** Condition。
        
    - 在节点完全准备好之前，`Ready` condition 的 `status` 可能会是 **`False`** 或 **`Unknown`**。
        
        - **`Unknown`**: 这通常发生在 API Server 在 `node-monitor-grace-period` (默认 40 秒) 内没有收到来自 kubelet 的心跳时。这可能意味着网络问题、kubelet 未运行或正在启动。
            
        - **`False`**: 表示 kubelet 正在运行，但节点存在某些问题，例如资源不足 (OutOfDisk, MemoryPressure, DiskPressure) 或网络配置问题 (NetworkUnavailable)，导致它无法接受 Pod。
            
4. **达到 `Ready` 状态：**
    
    - 当 kubelet 成功启动所有必要的组件（如容器运行时、kube-proxy 等），并且确认节点有足够的资源、网络连接正常时，它会将 `Ready` condition 的 `status` 更新为 **`True`**。
        
    - 一旦 `Ready` 状态变为 `True`，节点就被认为是健康的，并且可以开始接受由 Kubernetes 调度器分配的 Pods。
        

### 总结

所以，当节点刚注册到 API Server 时，它的状态可以概括为：

- API Server 中会创建一个对应的 **Node 对象**。
    
- 该 Node 对象的 **`Ready` Condition** 通常会先显示为 **`Unknown`** 或 **`False`**。
    
- 随着 kubelet 的初始化完成和系统组件的健康运行，它会最终将 `Ready` Condition 更新为 **`True`**。
    

你可以使用 `kubectl get nodes` 命令查看节点的概览状态，而 `kubectl describe node <node-name>` 命令则会提供更详细的节点状态和所有 Condition 信息。

