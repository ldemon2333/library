Below, we can see the representation of the HTTP API directory tree of Kubernetes:
![[Pasted image 20250509155316.png]]
HTTP API directory tree of Kubernetes can be divided into three independent group types:

- Core group **(/api/v1)**
	This group includes objects such as Pods, Services, Nodes, Namespaces, ConfigMaps, Secrets, etc.
- Named groupThis group includes objects in **/apis/$NAME/$VERSION** format. These different API versions imply different levels of stability and support:  
    - _Alpha level_ - it may be dropped at any point in time, without notice. For example, **/apis/batch/v2alpha1**.  
    - _Beta level_ - it is well-tested, but the semantics of objects may change in incompatible ways in a subsequent beta or stable release. For example, **/apis/certificates.k8s.io/v1beta1**.  
    - _Stable level_ - appears in released software for many subsequent versions. For example, **/apis/networking.k8s.io/v1**.
- System-wideThis group consists of system-wide API endpoints, like **/healthz**, **/logs**, **/metrics**, **/ui**, etc.


# k8s Object Model
Kubernetes 之所以流行，是因为其先进的应用程序生命周期管理功能。该功能通过丰富的对象模型实现，代表了 Kubernetes 集群中不同的持久化实体。这些实体描述了：
- What containerized applications we are running
- The nodes where the containerized applications are deployed
- Application resource consumption
- Policies attached to applications, like restart/upgrade policies, fault tolerance, ingress/egress, access control, etc.

几个字段
- spec
- apiVersion
- kind
- and additional data helpful to the cluster or users for accounting purposes - the **metadata**

In certain object definitions, however, we find different sections that replace **spec**, they are **data** and **stringData**. Both **data** and **stringData** sections facilitate the declaration of information that should be stored by their respective objects.

创建对象时，必须将 spec 字段下方的对象配置数据部分提交给 Kubernetes API 服务器。创建对象的 API 请求必须包含 spec 部分，用于描述所需状态以及其他详细信息。虽然 API 服务器接受 JSON 格式的对象定义，但我们通常以 YAML 格式提供此类定义清单，该清单由 kubectl 转换为 JSON 负载并发送到 API 服务器。

# Nodes
每个节点都由两个 Kubernetes 节点代理（kubelet 和 kube-proxy）进行管理，同时它还托管一个容器运行时。容器运行时需要运行节点上的所有容器化工作负载——控制平面代理和用户工作负载。kubelet 和 kube-proxy 节点代理负责执行所有与本地工作负载管理相关的任务——与运行时交互以运行容器、监控容器和节点的健康状况、向 API 服务器报告任何问题和节点状态，以及管理容器的网络流量。

根据其预定功能，节点分为两种类型：控制平面节点和工作节点。典型的 Kubernetes 集群至少包含一个控制平面节点，但为了实现控制平面的高可用性 (HA)，也可能包含多个控制平面节点。此外，集群还包含一个或多个工作节点，以在集群中提供资源冗余。在某些情况下，当高可用性和资源冗余并非至关重要时，单个一体化集群会作为单个虚拟机、裸机或容器上的单个节点引导启动。这些是混合节点，在同一系统上同时托管控制平面代理和用户工作负载。 Minikube 允许我们引导具有独立专用控制平面节点的多节点集群。但是，如果我们的主机系统物理资源（CPU、RAM、磁盘）有限，我们可以轻松地将单个一体化集群引导为单个虚拟机或容器上的单个节点，并且仍然可以探索本课程涵盖的大部分主题，但多节点集群特有的功能除外，例如 DaemonSet、多节点网络等。

节点标识在集群引导过程中由负责初始化集群代理的工具创建和分配。Minikube 使用默认的 kubeadm 引导工具，在 init 阶段初始化控制平面节点，并在 join 阶段通过添加工作节点或控制平面节点来扩展集群。

控制平面节点运行控制平面代理，例如 API 服务器、调度程序、控制器管理器和 etcd，此外还运行 kubelet 和 kube-proxy 节点代理、容器运行时以及用于容器网络、监控、日志记录、DNS 等的插件。

工作节点运行 kubelet 和 kube-proxy 节点代理、容器运行时以及用于容器网络、监控、日志记录、DNS 等的插件。

控制平面节点和工作节点共同构成了 Kubernetes 集群。集群的节点是分布在同一私有网络、不同网络甚至不同云网络上的系统。

# Namespaces
If multiple users and teams use the same k8s cluster we can partition the cluster into virtual subclusters using Namespaces. The names of the resources/objects created inside a Namespace are unique, but not across Namespaces in the cluster.

通常，Kubernetes 会创建四个开箱即用的命名空间：kube-system、kube-public、kube-node-lease 和 default。kube-system 命名空间包含 Kubernetes 系统创建的对象，主要是控制平面代理。default 命名空间包含管理员和开发人员创建的对象和资源，除非用户提供其他命名空间名称，否则默认会将对象分配给该命名空间。kube-public 是一个特殊的命名空间，它不安全且任何人都可读，用于特殊目的，例如公开集群的（非敏感）信息。最新的命名空间是 kube-node-lease，它包含用于节点心跳数据的节点租约对象。然而，良好的做法是根据需要创建其他命名空间，以虚拟化集群并隔离用户、开发团队、应用程序或层级：

# Pods
 A Pod is a logical collection of one or more containers, enclosing and isolating them to ensure that they:

- Are scheduled together on the same host with the Pod.
- Share the same network namespace, meaning that they share a single IP address originally assigned to the Pod.
- Have access to mount the same external storage (volumes) and other common dependencies.

![[Pasted image 20250509163526.png]]

Pod 本质上是短暂的，并且不具备自我修复的能力。因此，它们需要与控制器（或称 Operator，控制器/Operator 可互换使用）一起使用，控制器负责处理 Pod 的复制、容错、自我修复等功能。控制器的示例包括 Deployment、ReplicaSet、DaemonSet、Job 等。当使用 Operator 管理应用程序时，Pod 的规范会通过 Pod 模板嵌套在控制器的定义中。

containerPort 字段指定 Kubernetes 资源公开的容器端口，用于应用程序间访问或外部客户端访问——这将在“服务”章节中进行探讨。spec 的内容将用于调度目的，然后所选节点的 kubelet 将负责在该节点的容器运行时的帮助下运行容器镜像。Pod 的名称和标签用于工作负载核算。

编写定义清单（尤其是复杂的定义清单）可能会非常耗时且麻烦，因为 YAML 对缩进极其敏感。最终编辑此类定义清单时，请记住每个缩进的宽度为两个空格，并且应省略 TAB 键。

命令式地，我们可以简单地运行上面定义的 Pod，而不需要定义清单，如下所示：

**$ kubectl run nginx-pod --image=nginx:1.22.1 --port=80**

然而，当需要启动定义清单时，知道如何生成它可能会很有帮助。命令式命令加上额外的关键标志（例如 dry-run 和 yaml 输出），可以生成定义模板而不是运行 Pod，然后将模板存储在 nginx-pod.yaml 文件中。以下是一个多行命令，应完整选中进行复制/粘贴（包括反斜杠字符“\”）：

**$ kubectl run nginx-pod --image=nginx:1.22.1 --port=80 \  
--dry-run=client -o yaml > nginx-pod.yaml**

空运行，只是生成 yaml 文件

# Labels
标签是附加到 Kubernetes 对象（例如 Pod、ReplicaSet、Node、Namespace 和持久卷）的键值对。标签用于根据实际需求组织和选择对象子集。许多对象可以具有相同的标签。标签不提供对象的唯一性。控制器使用标签将解耦的对象逻辑分组，而不是使用对象的名称或 ID。

![[Pasted image 20250509165513.png]]
In the image above, we have used two Label keys: **app** and **env**. Based on our requirements, we have given different values to our four Pods. The Label **env=dev** logically selects and groups the top two Pods, while the Label **app=frontend** logically selects and groups the left two Pods. We can select one of the four Pods - bottom left, by selecting two Labels: **app=frontend** **AND** **env=qa**.

# label Selectors
Controllers, or operators, and Services, use [label selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors) to select a subset of objects. Kubernetes supports two types of Selectors:
**Equality-Based Selectors**

**Set-Based Selectors**

# ReplicationControllers
虽然不再是推荐的控制器，但 ReplicationController 是一个复杂的 operator，它通过不断将实际状态与托管应用程序的期望状态进行比较，确保在任何给定时间都有指定数量的 Pod 副本运行所需版本的应用程序容器。如果 Pod 数量超过期望数量，复制控制器会随机终止超过期望数量的 Pod；如果 Pod 数量少于期望数量，复制控制器会请求创建更多 Pod，直到实际数量与期望数量匹配。通常，我们不会独立部署 Pod，因为如果 Pod 因错误终止而无法自行重启，因为 Pod 缺少 Kubernetes 承诺的自我修复功能。推荐的方法是使用某种类型的运算符来运行和管理 Pod。

In addition to replication, the ReplicationController operator also supports application updates.

However, the default recommended controller is the [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) which configures a [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) controller to manage application Pods' lifecycle.

# ReplicaSets
A [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) is, in part, the next-generation ReplicationController, as it implements the replication and self-healing aspects of the ReplicationController. ReplicaSets support both equality- and set-based Selectors, whereas ReplicationControllers only support equality-based Selectors.

Pod 定义的应用程序的生命周期将由控制器 ReplicaSet 监控。借助 ReplicaSet，我们可以扩展运行特定应用程序容器镜像的 Pod 数量。扩展可以手动完成，也可以使用自动扩缩器。

在 Kubernetes 的 `ReplicaSet` 配置中，`selector` 字段用于 **指定该 ReplicaSet 应该管理哪些 Pod**。这是通过标签匹配实现的。

在你的配置中：

```yaml
selector:
  matchLabels:
    app: guestbook
```

这表示：**ReplicaSet 会查找标签为 `app: guestbook` 的 Pod 来进行管理。**

### 工作机制：

1. **匹配现有 Pod：** 如果当前集群中已经有一些 Pod，其标签匹配 `app: guestbook`，ReplicaSet 会“接管”它们。
    
2. **创建新 Pod：** 如果现有的匹配数量不足（比如配置了 `replicas: 3`，而匹配到的只有 1 个），它会根据 `template` 创建新 Pod 补足数量。
    
3. **不匹配的 Pod 不受控制：** ReplicaSet **不会**管理那些标签不匹配的 Pod。
    

### 注意事项：

- `selector.matchLabels` 中的标签 **必须匹配 template.metadata.labels**，否则 ReplicaSet 无法正确识别自己创建的 Pod，可能导致不断重建。
    
- 在你的例子中，selector 和 template 中的 labels 都是：
    
    ```yaml
    app: guestbook
    ```
    
    是一致的，配置是正确的。
    

ReplicaSets can be used independently as Pod controllers but they only offer a limited set of features. A set of complementary features are provided by Deployments, the recommended controllers for the orchestration of Pods. Deployments manage the creation, deletion, and updates of Pods. A Deployment automatically creates a ReplicaSet, which then creates a Pod. There is no need to manage ReplicaSets and Pods separately, the Deployment will manage them on our behalf.


# Deployments
Deployment 对象为 Pod 和 ReplicaSet 提供声明式更新。DeploymentController 是控制平面节点控制器管理器的一部分，作为控制器，它还确保当前状态始终与正在运行的容器化应用程序的期望状态匹配。它允许通过 rollout 和 rollback 实现无缝的应用程序更新和回滚（称为默认的 RollingUpdate 策略），并直接管理其 ReplicaSet 以实现应用程序的扩展。它还支持一种颠覆性的、不太流行的更新策略，称为 Recreate。
**apiVersion: apps/v1**  
**kind: Deployment**  
**metadata:**  
  **name: nginx-deployment**  
  **labels:**  
    **app: nginx-deployment**  
**spec:**  
  **replicas: 3**  
  **selector:**  
    **matchLabels:**  
      **app: nginx-deployment**  
  **template:**  
    **metadata:**  
      **labels:**  
        **app: nginx-deployment**  
    **spec:**  
      **containers:**  
      **- name: nginx**  
        **image: nginx:1.20.2**  
        **ports:**  
        **- containerPort: 80**

apiVersion 字段是第一个必填字段，它指定了我们要连接到的 API 服务器上的 API 端点；它必须与所定义对象类型的现有版本匹配。第二个必填字段是 kind，用于指定对象类型——在本例中是 Deployment，但也可以是 Pod、ReplicaSet、Namespace、Service 等。第三个必填字段 metadata 包含对象的基本信息，例如 name, annotations, labels, namespaces等。我们的示例展示了两个 spec 字段（spec 和 spec.template.spec）。第四个必填字段 spec 标志着定义 Deployment 对象期望状态的代码块的开始。在我们的示例中，我们请求在任何给定时间运行 3 个副本，即 Pod 的 3 个实例。Pod 是使用 spec.template 中定义的 Pod 模板创建的。嵌套对象（例如作为 Deployment 一部分的 Pod）将保留其元数据和 spec，并丢失其自身的 apiVersion 和 kind——两者都被模板替换。在 spec.template.spec 中，我们定义了 Pod 的期望状态。我们的 Pod 会创建一个单独的容器，运行来自 Docker Hub 的 nginx:1.20.2 镜像。

然而，当需要启动定义清单时，知道如何生成它可能会很有帮助。命令式命令加上额外的关键参数（例如 dry-run 和 yaml 输出），可以生成定义模板，而无需运行 Deployment，然后将模板存储在 nginx-deploy.yaml 文件中。以下是多行命令，应完整选中进行复制/粘贴（包括反斜杠“\”）：
**$ kubectl create deployment nginx-deployment \  
--image=nginx:1.20.2 --port=80 --replicas=3 \  
--dry-run=client -o yaml > nginx-deploy.yaml**

Once the Deployment object is created, the Kubernetes system attaches the **status** field to the object and populates it with all necessary status fields.

In the following example, a new **Deployment** creates **ReplicaSet** **A** which then creates **3 Pods**, with each Pod Template configured to run one **nginx:1.20.2** container image. In this case, the **ReplicaSet A** is associated with **nginx:1.20.2** representing a state of the **Deployment**. This particular state is recorded as **Revision 1**.
![[Pasted image 20250512223249.png]]
我们需要及时将更新推送到由 Deployment 对象管理的应用程序。让我们更改 Pod 的模板，并将容器镜像从 nginx:1.20.2 更新为 nginx:1.21.5。Deployment 会为版本号为 1.21.5 的新容器镜像触发一个新的 ReplicaSet B，此关联代表了 Deployment 的新记录状态，即修订版本 2。两个 ReplicaSet 之间的无缝过渡，即从包含三个版本号为 1.20.2 的 Pod 的 ReplicaSet A 到包含三个版本号为 1.21.5 的新 Pod 的 ReplicaSet B，或者从修订版本 1 到修订版本 2，都称为 Deployment 滚动更新。

当我们更新部署的 Pod 模板的特定属性时，会触发滚动更新。虽然计划内的变更（例如更新容器镜像、容器端口、卷和挂载）会触发新的修订版本，但其他动态操作（例如扩容或标记部署）不会触发滚动更新，因此不会更改修订版本号。

滚动更新完成后，部署将显示 ReplicaSet A 和 B，其中 A 已缩容为 0 个 Pod，B 已缩容为 3 个 Pod。这就是部署将其先前状态的配置设置记录为修订版本的方式。
![[Pasted image 20250512223519.png]]
一旦 ReplicaSet B 及其 3 个版本号为 1.21.5 的 Pod 准备就绪，Deployment 便会开始主动管理它们。然而，Deployment 会将其之前的配置状态保存为修订版本 (Revisions)，这对于 Deployment 的回滚功能（返回到之前已知的配置状态）至关重要。在我们的示例中，如果新的 nginx:1.21.5 的性能不令人满意，则可以将 Deployment 回滚到之前的修订版本，在本例中，从修订版本 2 回滚到修订版本 1，再次运行 nginx:1.20.2。

# DaemonSets
DaemonSets are operators designed to manage node agents. DaemonSet 在管理多个 Pod 副本和应用程序更新时与 ReplicaSet 和 Deployment 操作符类似，但 DaemonSet 具有一项独特的功能，即强制将单个 Pod 副本放置在每个节点、所有节点或选定的节点子集上。相比之下，ReplicaSet 和 Deployment 操作符默认无法控制同一节点上多个 Pod 副本的调度和放置。

DaemonSet 操作器通常用于从所有节点收集监控数据，或在所有节点上运行存储、网络或代理守护进程，以确保所有节点上始终运行特定类型的 Pod。它们是多节点 Kubernetes 集群中至关重要的 API 资源。在集群中每个节点上以 Pod 形式运行的 kube-proxy 代理，以及在集群所有节点上实现 Pod 网络的 Calico 或 Cilium 网络节点代理，都是由 DaemonSet 操作器管理的应用程序的示例。

每当一个节点添加到集群中时，来自指定 DaemonSet 的 Pod 都会自动放置在该节点上。尽管这确保了自动化过程，但 DaemonSet 的 Pod 是由控制器本身放置在集群所有节点上的，而不是借助默认调度程序。当任何一个节点崩溃或从集群中移除时，由 DaemonSet 管理的 Pod 都会被垃圾回收。如果删除了 DaemonSet，它创建的所有 Pod 副本也将被删除。

DaemonSet Pod 的放置仍然受调度属性控制，这些属性可能会限制其 Pod 仅放置在集群节点的子集上。这可以借助 Pod 调度属性（例如节点选择器、节点亲和性规则、污点和容忍度）来实现。这确保 DaemonSet 的 Pod 仅放置在特定节点上，例如根据需要放置在工作节点上。但是，如果启用了相应的功能，默认调度程序可以接管调度过程，并再次接受节点亲和性规则。


# Service
部署到 Kubernetes 集群的容器化应用程序可能需要访问其他类似的应用程序，或者需要能够被其他应用程序甚至客户端访问。这会带来问题，因为容器不会将其端口暴露给集群网络，而且也无法被发现。解决方案是像典型的容器主机那样进行简单的端口映射。然而，由于 Kubernetes 框架的复杂性，这种简单的端口映射并非易事。解决方案要复杂得多，它涉及 kube-proxy 节点代理、IP 表、路由规则、集群 DNS 服务器，所有这些共同实现了一种微负载均衡机制，该机制将容器的端口暴露给集群网络，甚至如果需要，还可以暴露给外部网络。这种机制称为服务 (Service)，它是将任何容器化应用程序暴露给 Kubernetes 网络的推荐方法。当暴露多副本应用程序（运行相同镜像的多个容器需要暴露相同端口）时，Kubernetes 服务的优势更加明显。在这种情况下，容器主机的简单端口映射将不再有效，但服务可以轻松实现如此复杂的需求。

这只是对 Kubernetes 服务资源的简要介绍。服务及其类型、配置选项等将在后续章节中讨论。