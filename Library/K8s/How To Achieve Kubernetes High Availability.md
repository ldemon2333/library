# High Availability for Kubernetes Control Plane
Kubernetes control plane has the following core components.

1. API server
2. Kube controller manager
3. Kube Scheduler
4. Cloud Controller Manager (Optional)

Running a single node control plane could lead to **single point of failure** of all the control plane components. To have a highly available Kubernetes control plane, you should have a minimum of **three quoram control plane nodes** with control plane components replicated across all the three nodes.
![[Pasted image 20250704172735.png]]
Now, its very important for you to understand the nature of each control plane component when deployed as multiple copies across nodes. Because few components use leader-election when deployed as multiple replicas.

## etcd
When it comes to etcd HA architecture, there are two modes.

1. **Stacked etcd:** etcd deployed along with control plane nodes
2. **External etcd cluster:** Etcd cluster running dedicated nodes. This model has the advantage of well-managed backup and restore options.

To have fault tolerance you should have a minimum of three node etcd cluster. The following table shows the fault tolerance of the etcd cluster.

|Cluster Size|Majority|Failure Tolerance|
|---|---|---|
|1|1|0|
|2|2|0|
|3|2|**1**|
|4|3|1|
|5|3|**2**|
|6|4|2|
|7|4|**3**|
etcd fault tolerance

When it comes to production deployments, it is essential to [backup etcd](https://devopscube.com/backup-etcd-restore-kubernetes/) periodically

## Api Server
The API server is a stateless application that primarily interacts with the etcd cluster to store and retrieve data. This means that multiple instances of the API server can be run across different control plane nodes.

To ensure that the cluster API is always available, a Load Balancer should be placed in front of the API server replicas. This Load Balancer endpoint is used by worker nodes, end-users, and external systems to interact with the cluster.

当您部署 Kubernetes API 服务器的多实例，并在它们前面放置一个负载均衡器时，**所有的 API 服务器实例都是同时工作的**。

---

### **多实例工作原理**

- **并行处理请求**: 负载均衡器接收到来自客户端（如 `kubectl`、工作节点、其他控制器）的请求后，会根据配置的负载均衡算法（例如**轮询 (Round Robin)**、**最少连接 (Least Connections)** 等）将请求分发给后端**所有健康的 API 服务器实例**。这意味着每个 API 服务器实例都在积极地接收和处理请求。
    
- **共享 etcd 后端**: 尽管有多个 API 服务器实例，但它们都连接到**同一个 etcd 集群**。etcd 负责存储集群的所有状态和数据，因此所有 API 服务器实例读写的数据都是一致的。API 服务器本身是**无状态**的，它们不存储任何持久化数据，只负责处理请求、验证身份、执行授权，并与 etcd 进行交互。
    
- **实现高可用性 (High Availability)**: 这种多实例部署模式是实现高可用性的关键。如果其中一个 API 服务器实例因为某种原因（例如节点故障、进程崩溃）变得不可用，负载均衡器会立即停止将请求转发给它，并自动将所有流量引导到剩余的健康实例。这确保了 Kubernetes API 的持续可用性，即使在部分组件发生故障时也不会中断。
    
- **扩展性**: 除了高可用性，多实例部署也提供了**水平扩展性**。当集群规模增大，API 请求量增加时，可以通过增加 API 服务器实例的数量来分散负载，提高 API 服务器的处理能力。

## kube Controller Manager
Kuber controller manager also follows the same leader-elections method. Out of many replicas, one controller manager is elected and leader and others are marked as followers. The leader controller is responsible for controlling the state of the cluster.

## Cloud Controller Manager
The Cloud Controller Manager (CCM) is a Kubernetes component that runs controllers that interact with cloud-provider-specific APIs to manage resources such as load balancers, persistent volumes, and routes.

Just like the scheduler and kube-controller, the CCM also uses leader election to ensure that there is only one active replica making decisions and interacting with the cloud-provider APIs at a time.

# High Availability for Worker Nodes
