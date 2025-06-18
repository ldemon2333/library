https://devopscube.com/kubernetes-architecture-explained/

The following **Kubernetes architecture diagram** shows all the components of the Kubernetes cluster and how external systems connect to the Kubernetes cluster.
![[Pasted image 20250615222626.png]]

# HTTP REST API
我们可以分几块来说清楚 **“HTTP REST API”** 这个说法。

---

### 🔹什么是 API？

API（Application Programming Interface，应用程序编程接口）是不同软件之间进行数据交换的一种方式。可以把它理解为**不同软件之间的一种“协议”**，说好怎样交换数据。

---

### 🔹什么是 REST？

REST（Representational State Transfer，表现层状态转移）是设计API时的一种**架构风格**，它有以下几个主要特点：

✅ **无状态（Stateless）**：每一次API调用是完全独立的，与之前发生的操作无关，**服务器不会保存客户端的会话**。

✅ **资源（Resources）**：RESTAPI将数据和功能抽象为“资源”，每种资源都有一个**唯一URL**。

✅ **表现（Representation）**：客户端可以按需要获取不同表现形式的数据（最常用的就是JSON或者XML）。

✅ **统一操作**：对资源进行操作时，主要依赖**HTTP提供的一致性的操作**（GET、POST、PUT、DELETE 等），而不是自己定义很多不同的“动作”。

---

### 🔹什么是 HTTP REST API？

**HTTP REST API** 就是**按REST架构风格设计出的API**，它**通过HTTP协议进行数据交换**。

简而言之：  
✅ **API**：对数据提供操作入口  
✅ **REST**：定义API如何进行设计（资源 + 无状态 + 一致操作）  
✅ **HTTP**：具体的数据传输协议（GET、POST、PUT、DELETE）

---

### 🔹举个例子：

假如我们有一个**“用户”**资源，API可以这样设计：

||URL|Method|作用|
|---|---|---|---|
|获取所有用户|`/users`|GET|列出全部用户|
|获取指定用户|`/users/{id}`|GET|获取指定 ID 的用户|
|新增用户|`/users`|POST|创建一个新用户|
|更新用户|`/users/{id}`|PUT|更新指定 ID 的用户|
|删除用户|`/users/{id}`|DELETE|删除指定 ID 的用户|

---

### 🔹数据交换样例（JSON）：

✅ **GET /users/123**

**响应**：

```json
{
  "id": 123,
  "name": "Alice",
  "email": "alice@example.com"
}
```

✅ **POST /users**

**请求主体（POST时发送的数据）**：

```json
{
  "name": "Charlie",
  "email": "charlie@example.com"
}
```

**响应**：

```json
{
  "id": 124,
  "name": "Charlie",
  "email": "charlie@example.com"
}
```

---

### 🔹总结：

✅ **API**：软件提供的数据和操作入口  
✅ **REST**：按资源进行组织，依赖HTTP进行操作，保持无状态  
✅ **HTTP**：具体的数据传输协议（GET、POST、PUT、DELETE）

So when you use kubectl to manage the cluster, at the backend you are actually communicating with the API server through **HTTP REST APIs**. However, the internal cluster components like the scheduler, controller, etc talk to the API server using [gRPC](https://grpc.io/docs/what-is-grpc/introduction/?ref=devopscube.com).

The communication between the API server and other components in the cluster happens over TLS to prevent unauthorized access to the cluster.

![[Pasted image 20250615223120.png]]

Kubernetes **api-server** is responsible for the following.

1. **API management**: Exposes the cluster API endpoint and handles all API requests. The API is version and itsupports multiple API versions simultaneously.
2. **Authentication** (Using client certificates, bearer tokens, and HTTP Basic Authentication) and **Authorization** (ABAC and RBAC evaluation)
3. Processing API requests and validating data for the API objects like pods, services, etc. (Validation and Mutation Admission controllers)
4. api-server coordinates all the processes between the control plane and worker node components.
5. API server also contianes an [aggreagation layer](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/?ref=devopscube.com) which allows you to extend Kubernetes API to create custom APIs resources and controllers.
6. The only component that the **kube-apiserver initiates a connection to** is the **etcd** component. All the other components connect to the API server.
7. The API server also **supports watching resources** for changes. For example, clients can establish a watch on specific resources and receive real-time notifications when those resources are created, modified, or deleted.
8. Each component (Kubelet, scheduler, controllers) **independently watches the API server** to figure out what it needs to do.


# 2 etcd
etcd uses [raft consensus algorithm](https://raft.github.io/?ref=devopscube.com) for strong consistency and availability. It works in a leader-member fashion for high availability and to withstand node failures.


下面是对 **Raft 一致性算法** 的简洁、直观介绍：

---

### 🔹Raft 是什么？

Raft 是为了**在分布式系统中实现数据副本高度一致**而设计的一种**一致性算法**。  
它可以让一组服务器（通常为 3 或 5 台）共同对操作进行**复制**（Replication），从而做到：  
✅ 宕机时数据不丢失  
✅ 系统对客户端表现为一个整体（对外是一个“领导者”）

---

### 🔹Raft 的核心思路：

➥ **领导者（Leader）**  
每时刻有且仅有一个领导者，负责处理客户端写操作，确保**数据按序复制到多数副本**。

➥ **跟随者（Follower）**  
被领导者复制数据，纯被动地进行响应。

➥ **候选者（Candidate）**  
若跟随者长时间收不到领导者心跳，会自己转为候选者，发起**选举**。

---

### 🔹Raft 一致性过程简述：

1️⃣ **领导者选举（Leader Election）**

- 服务器超时未收到领导者心跳时，会转为 Candidate，开始选举，自己给自己投一票，然后向其他节点求票。
    
- 一旦有**多数**（> 50%）节点投票给它，它就成为领导者。
    

2️⃣ **日志复制（Log Replication）**

- 由领导者接收客户端写操作，写到自己的操作日志中。
    
- 领导者异步地将此条日志复制到每一个跟随者。
    
- 一旦有**多数**跟随者确认写入，领导者就可以“提交”此条操作，并将结果应用到自己的“状态机”中。
    

---

### 🔹Raft 的优点：

✅ 简洁：结构和实现直观，容易理解  
✅ 容错：只要**大多数**节点是正常的，整体就能提供服务  
✅ 一致：通过领导者进行写操作序列化，保持数据副本高度一致


In a nutshell, here is what you need to know about etcd.
1. etcd stores all configurations, states, and metadata of Kubernetes objects (pods, secrets, daemonsets, [deployments](https://devopscube.com/kubernetes-deployment-tutorial/), configmaps, statefulsets, etc).
2. `etcd` allows a client to subscribe to events using `Watch()` API . Kubernetes api-server uses the etcd’s watch functionality to track the change in the state of an object.
3. etcd exposes key-value API [using gRPC](https://etcd.io/docs/v3.5/learning/api/?ref=devopscube.com). Also, the [gRPC gateway](https://etcd.io/docs/v3.3/dev-guide/api_grpc_gateway/?ref=devopscube.com) is a RESTful proxy that translates all the HTTP API calls into gRPC messages. This makes it an ideal database for Kubernetes.
4. etcd stores all objects under the **/registry** directory key in key-value format. For example, information on a pod named Nginx in the default namespace can be found under **/registry/pods/default/nginx**

![[Pasted image 20250615224159.png]]

Also, etcd it is the only **Statefulset** component in the control plane.

The number of nodes in an etcd cluster directly affects its fault tolerance. Here's how it breaks down:

1. **3 nodes**: Can tolerate 1 node failure (quorum = 2)
2. **5 nodes**: Can tolerate 2 node failures (quorum = 3)
3. **7 nodes**: Can tolerate 3 node failures (quorum = 4)

And so on. The general formula for the number of node failures a cluster can tolerate is:

```
fault tolerance = (n - 1) / 2
```

Where `n` is the total number of nodes.

# 3 kube-scheduler
The kube-scheduler is responsible for **scheduling Kubernetes pods on worker nodes**.

The following image shows a high-level overview of **how the scheduler works**.
![[Pasted image 20250615224428.png]]

In a Kubernetes cluster, there will be more than one worker node. So how does the scheduler select the node out of all worker nodes?

Here is how the scheduler works.

1. To choose the best node, the Kube-scheduler uses **filtering and scoring** operations.
2. In **filtering**, the scheduler finds the best-suited nodes where the pod can be scheduled. For example, if there are five worker nodes with resource availability to run the pod, it selects all five nodes. If there are no nodes, then the pod is unschedulable and moved to the scheduling queue. If It is a large cluster, let's say 100 worker nodes, and the scheduler doesn't iterate over all the nodes. There is a scheduler configuration parameter called **`percentageOfNodesToScore`**. The default value is typically **50%**. So it tries to iterate over 50% of nodes in a round-robin fashion. If the worker nodes are spread across multiple zones, then the scheduler iterates over nodes in different zones. For very large clusters the default **`percentageOfNodesToScore`** is 5%.
3. In the **scoring phase**, the scheduler ranks the nodes by assigning a score to the filtered worker nodes. The scheduler makes the scoring by calling multiple [scheduling plugins](https://kubernetes.io/docs/reference/scheduling/config/?ref=devopscube.com#scheduling-plugins). Finally, the worker node with the highest rank will be selected for scheduling the pod. If all the nodes have the same rank, a node will be selected at random.
4. Once the node is selected, the scheduler creates a binding event in the API server. Meaning an event to bind a pod and node.
![[Pasted image 20250615224900.png]]

Here is that you need to know about a scheduler.

1. It is a controller that listens to pod creation events in the API server.
2. The scheduler has two phases. **Scheduling cycle** and the **Binding cycle**. Together it is called the scheduling context. The scheduling cycle selects a worker node and the binding cycle applies that change to the cluster.
3. The scheduler always places the high-priority pods ahead of the low-priority pods for scheduling. Also, in some cases, after the pod starts running in the selected node, the pod might get evicted or moved to other nodes. If you want to understand more, read the [Kubernetes pod priority guide](https://devopscube.com/pod-priorityclass-preemption/)
4. You can create custom schedulers and run multiple schedulers in a cluster along with the native scheduler. When you deploy a pod you can specify the custom scheduler in the pod manifest. So the scheduling decisions will be taken based on the custom scheduler logic.
5. The scheduler has a [pluggable scheduling framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/?ref=devopscube.com). Meaning, that you can add your custom plugin to the scheduling workflow.

# 4 kube Controller Manager
**Kube controller manager** is a component that manages all the Kubernetes controllers. Kubernetes resources/objects like pods, namespaces, jobs, replicaset are managed by respective controllers. Also, the Kube scheduler is also a controller managed by the Kube controller manager.
![[Pasted image 20250615225317.png]]

# 5 Cloud Controller Manager (CCM)
When kubernetes is deployed in cloud environments, the cloud controller manager acts as a bridge between Cloud Platform APIs and the Kubernetes cluster.

This way the core kubernetes core components can work independently and allow the cloud providers to integrate with kubernetes using plugins. (For example, an interface between kubernetes cluster and AWS cloud API)

[Cloud controller integration](https://devopscube.com/aws-cloud-controller-manager/) allows Kubernetes cluster to provision cloud resources like instances (for nodes), Load Balancers (for services), and Storage Volumes (for persistent volumes).
![[Pasted image 20250615225635.png]]

Cloud Controller Manager contains a set of **cloud platform-specific controllers** that ensure the desired state of cloud-specific components (nodes, [Loadbalancers](https://devopscube.com/aws-load-balancers/), storage, etc). Following are the three main controllers that are part of the cloud controller manager.
1. **Node controller:** This controller updates node-related information by talking to the cloud provider API. For example, node labeling & annotation, getting hostname, CPU & memory availability, nodes health, etc.
2. **Route controller:** It is responsible for configuring networking routes on a cloud platform. So that pods in different nodes can talk to each other.
3. **Service controller**: It takes care of deploying load balancers for kubernetes services, assigning IP addresses, etc.

Following are some of the classic examples of cloud controller manager.

1. Deploying Kubernetes Service of type Load balancer. Here Kubernetes provisions a Cloud-specific Loadbalancer and integrates with Kubernetes Service.
2. Provisioning storage volumes (PV) for pods backed by cloud storage solutions.

Other than PodSpecs from the API server, kubelet can accept podSpec from a file, HTTP endpoint, and HTTP server. A good example of “podSpec from a file” is Kubernetes static pods.

Static pods are controlled by kubelet, not the API servers.

Here is a real-world example use case of the static pod.

While bootstrapping the control plane, kubelet starts the api-server, scheduler, and controller manager as static pods from podSpecs located at `/etc/kubernetes/manifests`

Following are some of the key things about kubelet.

1. Kubelet uses the CRI (container runtime interface) gRPC interface to talk to the container runtime.
2. It also exposes an HTTP endpoint to stream logs and provides exec sessions for clients.
3. Uses the CSI (container storage interface) gRPC to configure block volumes.
4. It uses the CNI plugin configured in the cluster to allocate the pod IP address and set up any necessary network routes and firewall rules for the pod.

![[Pasted image 20250615230310.png]]

# 2 Kube proxy
To understand Kube proxy, you need to have a basic knowledge of Kubernetes Service & endpoint objects.

Service in Kubernetes is a way to expose a set of pods internally or to external traffic. When you create the service object, it gets a virtual IP assigned to it. It is called clusterIP. It is only accessible within the Kubernetes cluster.

The Endpoint object contains all the IP addresses and ports of pod groups under a Service object. The endpoints controller is responsible for maintaining a list of pod IP addresses (endpoints). The service controller is responsible for configuring endpoints to a service.

You cannot ping the ClusterIP because it is only used for service discovery, unlike pod IPs which are pingable.

Now let's understand Kube Proxy.

Kube-proxy is a daemon that runs on every node as a [daemonset](https://devopscube.com/kubernetes-daemonset/). It is a proxy component that implements the Kubernetes Services concept for pods. (single DNS for a set of pods with load balancing). It primarily proxies UDP, TCP, and SCTP and does not understand HTTP.

When you expose pods using a Service (ClusterIP), Kube-proxy creates network rules to send traffic to the backend pods (endpoints) grouped under the Service object. Meaning, all the load balancing, and service discovery are handled by the Kube proxy.

Kube proxy talks to the API server to get the details about the Service (ClusterIP) and respective pod IPs & ports (endpoints). It also monitors for changes in service and endpoints.

Kube-proxy then uses any one of the following modes to create/update rules for routing traffic to pods behind a Service

1. **[IPTables](https://wiki.centos.org/HowTos/Network/IPTables?ref=devopscube.com)**: It is the default mode. In IPTables mode, the traffic is handled by IPtable rules. This means that for each service, IPtable rules are created. These rules capture the traffic coming to the ClusterIP and then forward it to the backend pods. Also, In this mode, kube-proxy chooses the backend pod random for load balancing. Once the connection is established, the requests go to the same pod until the connection is terminated.
2. **[IPVS](https://en.wikipedia.org/wiki/IP_Virtual_Server?ref=devopscube.com):** For clusters with services exceeding 1000, IPVS offers performance improvement. It supports the following load-balancing algorithms for the backend.
    1. `rr`: round-robin : It is the default mode.
    2. `lc`: least connection (smallest number of open connections)
    3. `dh`: destination hashing
    4. `sh`: source hashing
    5. `sed`: shortest expected delay
    6. `nq`: never queue
3. **Userspace** (legacy & not recommended)
4. **Kernelspace**: This mode is only for Windows systems.

![[Pasted image 20250615230835.png]]
If you would like to understand the performance difference between kube-proxy IPtables and IPVS mode, [read this article](https://www.tigera.io/blog/comparing-kube-proxy-modes-iptables-or-ipvs/?ref=devopscube.com).

Also, you can run a Kubernetes cluster without kube-proxy by replacing it with [Cilium](https://docs.cilium.io/en/v1.9/gettingstarted/kubeproxy-free/?ref=devopscube.com).

How does the CNI Plugin work with Kubernetes?

1. The Kube-controller-manager is responsible for assigning pod CIDR to each node. Each pod gets a unique IP address from the pod CIDR.
2. Kubelet interacts with container runtime to launch the scheduled pod. The CRI plugin which is part of the Container runtime interacts with the CNI plugin to configure the pod network.
3. CNI Plugin enables networking between pods spread across the same or different nodes using an overlay network.

![[Pasted image 20250615231832.png]]
