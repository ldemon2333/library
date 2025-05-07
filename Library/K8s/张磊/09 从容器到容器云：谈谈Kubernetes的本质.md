从这个结构中我们不难看出，一个正在运行的Linux容器，其实可以被“一分为二”地看待：

1. 一组联合挂载在/var/lib/docker/aufs/mnt上的rootfs，这一部分我们称为“容器镜像”（Container Image），是容器的静态视图；
    
2. 一个由Namespace+Cgroups构成的隔离环境，这一部分我们称为“容器运行时”（Container Runtime），是容器的动态视图。

![[Pasted image 20250506212821.png]]
其中，控制节点，即Master节点，由三个紧密协作的独立组件组合而成，它们分别是负责API服务的kube-apiserver、负责调度的kube-scheduler，以及负责容器编排的kube-controller-manager。整个集群的持久化数据，则由kube-apiserver处理后保存在Etcd中。

而计算节点上最核心的部分，则是一个叫作kubelet的组件。

**在Kubernetes项目中，kubelet主要负责同容器运行时（比如Docker项目）打交道**。而这个交互所依赖的，是一个称作CRI（Container Runtime Interface）的远程调用接口，这个接口定义了容器运行时的各项核心操作，比如：启动一个容器需要的所有参数。

这也是为何，Kubernetes项目并不关心你部署的是什么容器运行时、使用的什么技术实现，只要你的这个容器运行时能够运行标准的容器镜像，它就可以通过实现CRI接入到Kubernetes项目当中。

而具体的容器运行时，比如Docker项目，则一般通过OCI这个容器运行时规范同底层的Linux操作系统进行交互，即：把CRI请求翻译成对Linux操作系统的调用（操作Linux Namespace和Cgroups等）。

**此外，kubelet还通过gRPC协议同一个叫作Device Plugin的插件进行交互**。这个插件，是Kubernetes项目用来管理GPU等宿主机物理设备的主要组件，也是基于Kubernetes项目进行机器学习训练、高性能作业支持等工作必须关注的功能。

而**kubelet的另一个重要功能，则是调用网络插件和存储插件为容器配置网络和持久化存储**。这两个插件与kubelet进行交互的接口，分别是CNI（Container Networking Interface）和CSI（Container Storage Interface）。

运行在大规模集群中的各种任务之间，实际上存在着各种各样的关系。这些关系的处理，才是作业编排和管理系统最困难的地方。

事实也正是如此。

其实，这种任务与任务之间的关系，在我们平常的各种技术场景中随处可见。比如，一个Web应用与数据库之间的访问关系，一个负载均衡器和它的后端服务之间的代理关系，一个门户应用与授权组件之间的调用关系。

更进一步地说，同属于一个服务单位的不同功能之间，也完全可能存在这样的关系。比如，一个Web应用与日志搜集组件之间的文件交换关系。

而在容器技术普及之前，传统虚拟机环境对这种关系的处理方法都是比较“粗粒度”的。你会经常发现很多功能并不相关的应用被一股脑儿地部署在同一台虚拟机中，只是因为它们之间偶尔会互相发起几个HTTP请求。

更常见的情况则是，一个应用被部署在虚拟机里之后，你还得手动维护很多跟它协作的守护进程（Daemon），用来处理它的日志搜集、灾难恢复、数据备份等辅助工作。

但容器技术出现以后，你就不难发现，在“功能单位”的划分上，容器有着独一无二的“细粒度”优势：毕竟容器的本质，只是一个进程而已。

也就是说，只要你愿意，那些原先拥挤在同一个虚拟机里的各个应用、组件、守护进程，都可以被分别做成镜像，然后运行在一个个专属的容器中。它们之间互不干涉，拥有各自的资源配额，可以被调度在整个集群里的任何一台机器上。而这，正是一个PaaS系统最理想的工作状态，也是所谓“微服务”思想得以落地的先决条件。

当然，如果只做到“封装微服务、调度单容器”这一层次，Docker Swarm项目就已经绰绰有余了。如果再加上Compose项目，你甚至还具备了处理一些简单依赖关系的能力，比如：一个“Web容器”和它要访问的数据库“DB容器”。

在Compose项目中，你可以为这样的两个容器定义一个“link”，而Docker项目则会负责维护这个“link”关系，其具体做法是：Docker会在Web容器中，将DB容器的IP地址、端口等信息以环境变量的方式注入进去，供应用进程使用，比如：

```
DB_NAME=/web/db DB_PORT=tcp://172.17.0.5:5432 DB_PORT_5432_TCP=tcp://172.17.0.5:5432 DB_PORT_5432_TCP_PROTO=tcp DB_PORT_5432_TCP_PORT=5432 DB_PORT_5432_TCP_ADDR=172.17.0.5
```

而当DB容器发生变化时（比如，镜像更新，被迁移到其他宿主机上等等），这些环境变量的值会由Docker项目自动更新。**这就是平台项目自动地处理容器间关系的典型例子。**

可是，如果我们现在的需求是，要求这个项目能够处理前面提到的所有类型的关系，甚至还要能够支持未来可能出现的更多种类的关系呢？

这时，“link”这种单独针对一种案例设计的解决方案就太过简单了。如果你做过架构方面的工作，就会深有感触：一旦要追求项目的普适性，那就一定要从顶层开始做好设计。

所以，**Kubernetes项目最主要的设计思想是，从更宏观的角度，以统一的方式来定义任务之间的各种关系，并且为将来支持更多种类的关系留有余地。**

比如，Kubernetes项目对容器间的“访问”进行了分类，首先总结出了一类非常常见的“紧密交互”的关系，即：这些应用之间需要非常频繁的交互和访问；又或者，它们会直接通过本地文件进行信息交换。

在常规环境下，这些应用往往会被直接部署在同一台机器上，通过Localhost通信，通过本地磁盘目录交换文件。而在Kubernetes项目中，这些容器则会被划分为一个“Pod”，Pod里的容器共享同一个Network Namespace、同一组数据卷，从而达到高效率交换信息的目的。

而对于另外一种更为常见的需求，比如Web应用与数据库之间的访问关系，Kubernetes项目则提供了一种叫作“Service”的服务。像这样的两个应用，往往故意不部署在同一台机器上，这样即使Web应用所在的机器宕机了，数据库也完全不受影响。可是，我们知道，对于一个容器来说，它的IP地址等信息不是固定的，那么Web应用又怎么找到数据库容器的Pod呢？

所以，Kubernetes项目的做法是给Pod绑定一个Service服务，而Service服务声明的IP地址等信息是“终生不变”的。这个**Service服务的主要作用，就是作为Pod的代理入口（Portal），从而代替Pod对外暴露一个固定的网络地址**。

这样，对于Web应用的Pod来说，它需要关心的就是数据库Pod的Service信息。不难想象，Service后端真正代理的Pod的IP地址、端口等信息的自动更新、维护，则是Kubernetes项目的职责。

像这样，围绕着容器和Pod不断向真实的技术场景扩展，我们就能够摸索出一幅如下所示的Kubernetes项目核心功能的“全景图”。

![[Pasted image 20250506213855.png]]
按照这幅图的线索，我们从容器这个最基础的概念出发，首先遇到了容器间“紧密协作”关系的难题，于是就扩展到了Pod；有了Pod之后，我们希望能一次启动多个应用的实例，这样就需要Deployment这个Pod的多实例管理器；而有了这样一组相同的Pod后，我们又需要通过一个固定的IP地址和端口以负载均衡的方式访问它，于是就有了Service。

**除了应用与应用之间的关系外，应用运行的形态是影响“如何容器化这个应用”的第二个重要因素。**

为此，Kubernetes定义了新的、基于Pod改进后的对象。比如Job，用来描述一次性运行的Pod（比如，大数据任务）；再比如DaemonSet，用来描述每个宿主机上必须且只能运行一个副本的守护进程服务；又比如CronJob，则用于描述定时任务等等。

如此种种，正是Kubernetes项目定义容器间关系和形态的主要方法。

可以看到，Kubernetes项目并没有像其他项目那样，为每一个管理功能创建一个指令，然后在项目中实现其中的逻辑。这种做法，的确可以解决当前的问题，但是在更多的问题来临之后，往往会力不从心。

相比之下，在Kubernetes项目中，我们所推崇的使用方法是：

- 首先，通过一个“编排对象”，比如Pod、Job、CronJob等，来描述你试图管理的应用；
- 然后，再为它定义一些“服务对象”，比如Service、Secret、Horizontal Pod Autoscaler（自动水平扩展器）等。这些对象，会负责具体的平台级功能。

**这种使用方法，就是所谓的“声明式API”。这种API对应的“编排对象”和“服务对象”，都是Kubernetes项目中的API对象（API Object）。**

最后，我来回答一个更直接的问题：Kubernetes项目如何启动一个容器化任务呢？

比如，我现在已经制作好了一个Nginx容器镜像，希望让平台帮我启动这个镜像。并且，我要求平台帮我运行两个完全相同的Nginx副本，以负载均衡的方式共同对外提供服务。

- 如果是自己DIY的话，可能需要启动两台虚拟机，分别安装两个Nginx，然后使用keepalived为这两个虚拟机做一个虚拟IP。
    
- 而如果使用Kubernetes项目呢？你需要做的则是编写如下这样一个YAML文件（比如名叫nginx-deployment.yaml）：
    
在上面这个YAML文件中，我们定义了一个Deployment对象，它的主体部分（spec.template部分）是一个使用Nginx镜像的Pod，而这个Pod的副本数是2（replicas=2）。

然后执行：

```lua
$ kubectl create -f nginx-deployment.yaml
```

这样，两个完全相同的Nginx容器副本就被启动了。

不过，这么看来，做同样一件事情，Kubernetes用户要做的工作也不少嘛。

# 总结
首先，我和你一起回顾了容器的核心知识，说明了容器其实可以分为两个部分：容器运行时和容器镜像。

然后，我重点介绍了Kubernetes项目的架构，详细讲解了它如何使用“声明式API”来描述容器化业务和容器间关系的设计思想。

实际上，过去很多的集群管理项目（比如Yarn、Mesos，以及Swarm）所擅长的，都是把一个容器，按照某种规则，放置在某个最佳节点上运行起来。这种功能，我们称为“调度”。

而Kubernetes项目所擅长的，是按照用户的意愿和整个系统的规则，完全自动化地处理好容器之间的各种关系。**这种功能，就是我们经常听到的一个概念：编排。**

所以说，Kubernetes项目的本质，是为用户提供一个具有普遍意义的容器编排工具。

不过，更重要的是，Kubernetes项目为用户提供的不仅限于一个工具。它真正的价值，乃在于提供了一套基于容器构建分布式系统的基础依赖。关于这一点，相信你会在今后的学习中，体会越来越深。


1. 在Kubernetes之前，很多项目都没办法管理“有状态”的容器，即，不能从一台宿主机“迁移”到另一台宿主机上的容器。你是否能列举出，阻止这种“迁移”的原因都有哪些呢？

你的问题非常关键，涉及 Kubernetes 出现之前容器编排系统面临的核心挑战，尤其是在处理“有状态容器（**stateful containers**）”时的难题。

---

## 🧠 什么是“有状态容器”（Stateful Container）

**有状态容器**是指：容器在运行过程中会**存储持久化的数据**，比如数据库的内容、缓存、文件、用户上传等。一旦这个容器崩溃或迁移到其他宿主机，这些数据**必须保留且保持一致性**，否则就会丢失业务状态。

常见的有状态容器示例：

- MySQL / PostgreSQL / MongoDB 等数据库容器
    
- Redis、Kafka 等有持久化能力的服务
    
- 存储用户上传的文件、缓存中间结果的服务
    

---

## ❌ 在 Kubernetes 出现之前，阻止容器“迁移”的主要原因

|原因|解释|
|---|---|
|❌ 持久化存储机制缺失|Docker 原生的 volume 是本地绑定（hostPath），数据跟主机绑定，迁移容器时数据就“断了”。|
|❌ 缺乏统一的存储抽象层|不同主机的存储方案各异，没有统一接口进行挂载和访问。|
|❌ IP 和网络不固定|容器迁移到其他节点，IP 会变，连接断裂，客户端无法访问。|
|❌ 数据一致性难以保障|容器之间共享状态数据难以同步，存在并发写入、数据冲突等问题。|
|❌ 缺乏有状态服务编排支持|如 Docker Swarm、Mesos 等早期系统以无状态服务为主，缺乏 StatefulSet、PersistentVolume 的概念。|
|❌ 手动管理卷挂载复杂|卷绑定需要自己管理、拷贝、恢复快照，运维复杂且容易出错。|
|❌ 缺乏“存储与容器生命周期解耦”能力|数据卷与容器生命周期强绑定，删除容器就可能导致数据也被删除。|

---

## ✅ Kubernetes 如何解决这些问题

|问题|Kubernetes 解决方案|
|---|---|
|持久存储|提供 **PersistentVolume（PV）** 和 **PersistentVolumeClaim（PVC）** 抽象，支持本地盘、NFS、Ceph、EBS 等多种后端|
|可迁移性|容器挂载 PV 时是通过 CSI 接口动态挂载的，可以在多个节点之间迁移|
|网络持久化|配合 **StatefulSet** 保证 DNS 名字、存储卷、顺序启动等一致|
|数据一致性|StatefulSet 支持稳定标识符，确保服务端有序启动、挂载对应的存储|
|自动恢复|配合 operator、controller，系统崩溃后自动重建容器、挂载卷、恢复服务|

好的，我们来举一个在 Kubernetes 中部署有状态容器 —— **MySQL 数据库** 的例子。这是最常见的“有状态服务”场景之一。

---

## ✅ 用 Kubernetes 部署有状态 MySQL（使用 StatefulSet）

### 📁 1. 创建 PersistentVolume 和 PersistentVolumeClaim

```yaml
# mysql-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/mysql-pv   # 注意：真实生产中应改用动态存储，如 NFS、EBS 等
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

---

### 🐋 2. 创建 MySQL 的 StatefulSet（或 Deployment）

```yaml
# mysql-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql"
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: my-secret-pw
          ports:
            - containerPort: 3306
              name: mysql
          volumeMounts:
            - name: mysql-storage
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: mysql-storage
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

---

### 📡 3. 创建对应的 Headless Service（用于稳定 DNS）

```yaml
# mysql-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None   # Headless Service
  selector:
    app: mysql
  ports:
    - port: 3306
```

---

## 💡 解读这个例子中体现的“有状态支持”

|特性|实现方式|
|---|---|
|数据持久化|使用 PVC + PV 或 VolumeClaimTemplate 自动绑定卷|
|容器崩溃后数据保留|由于数据写入 `/var/lib/mysql`，即使容器重建，卷还在，数据不丢失|
|容器迁移到其他节点|Kubernetes 会重新挂载数据卷（如使用 NFS、EBS 等）并恢复服务|
|网络标识不变|StatefulSet 提供稳定 DNS，例如 `mysql-0.mysql.default.svc.cluster.local`|

Headless Service（无头服务）是 Kubernetes 中一种特殊类型的 Service，**没有 ClusterIP（即没有负载均衡功能）**，其作用主要是为了让客户端能够**直接发现并访问后端每个 Pod 的 IP 或 DNS 名字**，而不是通过一个统一的 IP 地址。

---

## 🔍 什么是 Headless Service？

正常的 Service（如 `ClusterIP`）会：

- 为服务分配一个虚拟 IP（ClusterIP）
    
- 对后端 Pod 做负载均衡（Round-Robin）
    
- 客户端访问 Service IP → Kubernetes 代理请求到某个 Pod
    

而 **Headless Service** 设置了：

```yaml
spec:
  clusterIP: None
```

这会：

- **不分配 ClusterIP**
    
- 不做负载均衡
    
- 客户端可以通过 DNS 查询到后端 **所有 Pod 的 IP 地址列表或具体 DNS 名字**
    

---

## 📦 示例：StatefulSet + Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None         # 👈 关键点：Headless Service
  selector:
    app: mysql
  ports:
    - port: 3306
```

配合 StatefulSet 后：

Kubernetes 会自动创建 DNS：

```text
mysql-0.mysql.default.svc.cluster.local
mysql-1.mysql.default.svc.cluster.local
```

这样，每个 Pod 都有**固定的 DNS 名字**，便于服务互联，比如：

- ZooKeeper 节点通信
    
- MySQL 主从集群
    
- Kafka Broker 集群
    

---

## ✅ Headless Service 的应用场景

|场景|说明|
|---|---|
|StatefulSet|每个 Pod 都有独立身份，不能用负载均衡|
|集群服务发现|比如 Kafka、ZooKeeper、Redis Cluster|
|DNS A 记录暴露多个 Pod IP|比如客户端做自己策略的负载均衡|

---

## 🎯 总结一句话

> **Headless Service 是为了让客户端“看见”后端的每一个 Pod，而不是被统一的服务 IP 隐藏起来。**

---

如果你想测试 Headless Service 的 DNS 效果，我可以提供 `nslookup` 和 `dig` 的示例命令。是否需要？