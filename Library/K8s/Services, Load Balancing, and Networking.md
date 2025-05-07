service 定义了 Pod 的逻辑集合和访问该集合的策略，是真实服务的抽象。Service 提供了一个统一的服务访问入口以及服务代理和发现机制。用户不需要了解后台Pod是如何运行。只需要将一组跑同一服务的pod池化成一个service，k8s集群会自动给这个service分配整个集群唯一ip和端口号（这个端口号自己在yaml文件中定义），一个service定义了访问pod的方式，就像单个固定的IP地址和与其相对应的DNS名之间的关系。

Service 是可以为一组具有相同功能的容器应用提供统一的入口地址，并且将请求进行负载分发到后端的各个容器应用上。

service 有以下三种类型：
- ClusterIP：只能提供K8S集群内可以访问的IP和Port
- NodePort：在上面的基础上，提供K8S集群外部可以访问的Port，IP则是集群内任意一个NodeIP。
- LoadBalancer：在上面的基础上，提供K8S集群外可以访问的IP和Port。需要注意的是，在LoadBlancer中可以设置关闭NodePort。还有一点需要注意，LoadBlanacer依赖集群底座的能力，并不是所有的K8S都能使用LoadBlancer。

![[Pasted image 20250418221007.png]]

集群内部可以直接通过service的ip+端口访问，而nodeport就是为了外网访问服务，给node开了一个端口，转发到service的ip+端口。




# The k8s network model
- Each pod in a cluster gets its own unique cluster-wide IP address.
	- A pod has its own private network namespace which is shared by all of the containers within the pod. Processers running in different containers in the same pod can communicate with each other over `localhost`.
- The pod network
	- All pods can communicate with all other pods, whether they are on the same node or on different nodes. Pods can communicate with each other directly, without the use of proxies or address translation (NAT).
	- Agents on a node (kubelet) can communicate with all pods on that node.

**Kubernetes 的 Service API** 允许你为一个由一个或多个后端 Pod 实现的服务，提供一个**稳定（长生命周期）的 IP 地址或主机名，即使这些 Pod 会随着时间推移而变化（例如重启、伸缩等）。

这段英文描述的是 **Kubernetes 中 Service 与流量转发之间的内部机制**，特别强调了 **EndpointSlice 与 Service Proxy（如 kube-proxy）** 的协作。我们可以逐句翻译，并附上解释帮助你更深入理解：


> **Kubernetes 会自动管理 EndpointSlice 对象，以提供当前为某个 Service 提供服务的 Pod 的信息。**

💡 解释：

- EndpointSlice 是对传统 Endpoints 的升级（自 v1.21 起默认启用），用于表示一个 Service 的后端 Pod 信息，包含 Pod 的 IP、端口、健康状态等；
    
- 每个 Service 对应一组 EndpointSlice，Kubernetes 控制器会在 Pod 状态变化时自动更新它们。


> **一个 Service 代理的实现（如 kube-proxy）会监听 Service 和 EndpointSlice 对象的变化，并据此编排数据平面，将服务流量路由到后端 Pod。它通常通过操作系统或云提供商的 API 来实现包的拦截或重写。**


- kube-proxy 就是一个 **Service Proxy 的实现**，运行在每个节点上；
    
- 它会监控 API Server 中 Service 与 EndpointSlice 的变化（通过 watch）；
    
- 根据这些信息，它使用系统 API（如 iptables、ipvs、或 eBPF）来控制本节点的网络转发行为；
    
- 例如使用 iptables 的 DNAT，将目标 IP 和端口从 Service 重写为某个后端 Pod 的 IP 和端口；
    
- 在云环境中，它也可能调用云厂商提供的 Load Balancer API 来实现外部访问能力。
    


### 🔁 总结串联：

Kubernetes 的 Service 资源本身**并不负责转发流量**，它的作用是定义服务入口（IP + 端口 + 选择器），实际的流量转发是通过 **kube-proxy（或其他代理实现）** 完成的：

1. 控制面自动维护 **EndpointSlice**（后端 Pod 列表）；
    
2. kube-proxy 等代理在每个节点监听这些对象的变化；
    
3. 它们更新本地的网络规则（iptables、IPVS、eBPF等）；
    
4. 当有客户端访问 Service IP 时，操作系统会根据这些规则把流量转发给真实的 Pod。

- The Gateway API (or its predecessor, Ingress) allows you to make Services accessible to clients that are outside the cluster.

- NetworkPolicy is a built-in K8s API that allows you to control traffic between pods, or between pods and the outside world.
