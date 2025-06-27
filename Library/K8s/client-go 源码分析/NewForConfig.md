```
clientset, err := kubernetes.NewForConfig(config)
```

![[Pasted image 20250626222005.png]]

在网络编程和系统设计中，特别是在 API 网关、微服务、客户端库等场景下，**QPS (Queries Per Second)** 和 **Burst (突发)** 是两个用于描述**流量限制（Rate Limiting）**和**流量整形（Traffic Shaping）**的关键概念。

当你说 "If config's RateLimiter is not set and QPS and Burst are acceptable" 时，这通常意味着：

1. **`config.RateLimiter` 未设置：** 这表示系统没有显式地启用或配置一个复杂的限流器。这可能意味着没有采用令牌桶、漏桶等算法来严格控制流量。
2. **QPS 和 Burst 是可接受的：** 尽管没有显式限流器，但系统对每秒的请求数 (QPS) 和短时间内的请求峰值 (Burst) 有一个**隐式的或默认的容忍度**。在这个容忍度内，系统可以正常处理请求。一旦超出这个容忍度，即使没有显式限流器，也可能导致性能下降、错误增加甚至系统崩溃。

---

### 1. QPS (Queries Per Second) - 每秒查询数

- **定义：** QPS 是衡量一个系统在**单位时间（通常是秒）内能够处理的请求数量**的指标。它代表了系统处理请求的**平均速率**。
- **作用：** 在限流语境中，QPS 设置了一个**长期平均速率上限**。如果你的 QPS 限制是 100，这意味着系统期望在较长时间段内，每秒接收的请求不会超过 100 个。
- **特点：** QPS 关注的是“稳态”或“平均”的流量。它不擅长处理短时间内的流量高峰。

举例：

如果一个 API Gateway 的 QPS 限制是 100，那么在 10 秒内，它理想情况下可以处理 1000 个请求（100 QPS * 10 秒）。如果在这 10 秒内请求均匀分布，例如每秒 100 个，那么一切正常。

---

### 2. Burst (突发)

- **定义：** Burst 指的是在**短时间内系统能够额外处理的、超出平均 QPS 限制的请求数量**。它代表了系统应对**流量峰值的能力**。
- **作用：** Burst 弥补了纯粹 QPS 限制的不足。如果没有 Burst 机制，即使在总请求量没有超过 QPS 限制的情况下，一个短时间的流量高峰（例如，某个时间点突然来了 500 个请求）也可能导致大量请求被拒绝或系统过载。Burst 允许系统在一定程度上“吸收”这些瞬时高峰。
- **特点：** Burst 关注的是“瞬时”或“峰值”的流量。它允许请求在一定程度上“超前”于平均速率被处理，然后在后续时间进行“填充”或“平滑”。

常见的实现机制：

Burst 通常与**令牌桶（Token Bucket）** 算法紧密相关。

- **令牌生成速率：** 令牌桶以一个固定的速率（通常等于 QPS）往桶里放入令牌。
- **桶的容量：** 桶的容量就是 Burst 的值。
- **请求处理：** 每个请求需要消耗一个令牌。如果桶里有足够的令牌，请求就可以被处理。
- **突发处理：** 当有突发流量时，只要桶里还有剩余的令牌，系统就可以立即处理这些请求，直到令牌用完。这允许系统在短时间内以高于 QPS 的速率处理请求。

举例：

假设 QPS = 100，Burst = 200。

- 系统每秒会生成 100 个令牌。
- 令牌桶最大可以存放 200 个令牌。
- 如果在某一秒内突然来了 300 个请求：
    - 如果有 200 个令牌在桶里（因为之前流量很低，令牌积累起来了），那么这 200 个请求可以立即被处理。
    - 剩下的 100 个请求则需要等待新的令牌生成（或者被拒绝，取决于具体的限流策略）。
- 这 200 个突发请求就是通过 Burst 容量来处理的。

---
### QPS 和 Burst 的关系与重要性
- **防止雪崩：** 在分布式系统中，限流是防止“雪崩效应”的关键措施。通过限制进入系统的请求量，可以避免单个服务过载导致整个系统崩溃。

# 深拷贝与浅拷贝
在编程中，**浅拷贝（Shallow Copy）** 和 **深拷贝（Deep Copy）** 是两种不同的复制对象的方式，它们在处理对象的引用类型字段时表现出关键差异。理解这两种拷贝方式对于避免程序中的意外行为至关重要，尤其是在涉及到复杂数据结构（如嵌套对象、切片、映射等）时。

---

### 1. 浅拷贝（Shallow Copy）

当进行浅拷贝时，会创建一个新对象，并将原始对象的所有字段值复制到新对象中。但是，对于**引用类型（如指针、切片、映射、结构体中的引用字段等）的字段，浅拷贝只会复制引用本身（即内存地址）**，而不会复制引用所指向的实际数据。

这意味着：

- **值类型字段：** 如果字段是基本类型（如整数、浮点数、布尔值、字符串等），它们的值会被直接复制到新对象中。修改新对象中的这些字段不会影响原始对象。
- **引用类型字段：** 新对象和原始对象中的引用类型字段将**指向同一块内存地址**。因此，如果通过新对象修改了这些引用类型字段所指向的数据，原始对象也会受到影响，反之亦然。

假设我们有一个结构体 `Config` 包含一个 `RateLimiter` 字段，这个字段本身是一个指向 `RateLimiter` 对象的指针或结构体。

```
type RateLimiter struct {
	QPS int
	Burst int
}

type Config struct {
	Name string
	Limiter *RateLimiter // Limiter 是一个指针类型
}

// 原始对象
originalConfig := Config{
	Name: "ServiceA",
	Limiter: &RateLimiter{QPS: 10, Burst: 20},
}

// 浅拷贝：直接赋值或某些库的浅拷贝方法
configShallowCopy := originalConfig

// 修改浅拷贝的 Limiter 字段指向的数据
configShallowCopy.Limiter.QPS = 5 // !!! 原始对象的 Limiter.QPS 也会变成 5

fmt.Println(originalConfig.Limiter.QPS) // 输出 5
```

在这个例子中，`configShallowCopy` 和 `originalConfig` 都拥有一个 `Limiter` 字段，并且这两个 `Limiter` 字段**都指向内存中的同一个 `RateLimiter` 实例**。所以，通过 `configShallowCopy` 修改 `Limiter` 的内容，`originalConfig` 也会看到这些变化。

---

### 2. 深拷贝（Deep Copy）

当进行深拷贝时，不仅会复制原始对象的所有字段值，对于**引用类型**的字段，它会**递归地创建新的对象来复制这些引用所指向的实际数据**。

这意味着：

- **值类型字段：** 与浅拷贝相同，值类型字段会被直接复制。
- **引用类型字段：** 新对象中的引用类型字段将指向**新分配的内存空间中存储的复制数据**，而不是原始对象所指向的内存。因此，新对象和原始对象在内存中是完全独立的。修改新对象中的任何字段（包括引用类型字段指向的数据）都不会影响原始对象，反之亦然。

```
type RateLimiter struct {
	QPS int
	Burst int
}

type Config struct {
	Name string
	Limiter *RateLimiter
}

// 原始对象
originalConfig := Config{
	Name: "ServiceA",
	Limiter: &RateLimiter{QPS: 10, Burst: 20},
}

// 深拷贝：需要手动复制引用类型字段指向的对象
configDeepCopy := Config{
	Name: originalConfig.Name,
}
if originalConfig.Limiter != nil {
	configDeepCopy.Limiter = &RateLimiter{ // 创建一个新的 RateLimiter 实例
		QPS: originalConfig.Limiter.QPS,
		Burst: originalConfig.Limiter.Burst,
	}
}

// 修改深拷贝的 Limiter 字段指向的数据
configDeepCopy.Limiter.QPS = 5 // 原始对象的 Limiter.QPS 不会改变

fmt.Println(originalConfig.Limiter.QPS) // 输出 10
```

在这个例子中，`configDeepCopy.Limiter` 是一个**全新的 `RateLimiter` 实例**，它和 `originalConfig.Limiter` 指向的是不同的内存地址。所以，对 `configDeepCopy.Limiter` 的修改不会影响 `originalConfig.Limiter`。

---

**总结：**

- **浅拷贝**：只复制值的引用，不复制值本身。修改拷贝会影响原数据。
- **深拷贝**：递归复制所有值，包括引用类型所指向的数据。修改拷贝不会影响原数据。

# AdmissionregistrationV1Client
### Kubernetes API Group (`admissionregistration.k8s.io`) 解释

在 Kubernetes 中，API 资源是按照**API 组（API Group）**来组织的。API 组的引入是为了：

- **模块化和扩展性：** 将相关的资源类型（Kind）组织在一起，使得 Kubernetes API 更易于管理和扩展。
    
- **版本控制：** 不同的 API 组可以独立地进行版本迭代，例如 `apps/v1` 和 `batch/v1` 可以有不同的发布周期。
    
- **权限控制：** 可以基于 API 组来定义 RBAC 权限，例如允许某个用户管理 `apps` 组的资源，但不允许管理 `batch` 组的资源。
    
- **避免命名冲突：** 不同的组可以使用相同的资源名称（Kind），例如 `Deployment` 既可以存在于 `apps` 组（`apps/v1.Deployment`），也可以在 `extensions` 组（`extensions/v1beta1.Deployment`，尽管后者已废弃），通过 API 组进行区分。
    

`admissionregistration.k8s.io` 就是 Kubernetes 中一个非常重要的 API 组，它包含了用于**准入控制（Admission Control）**的相关资源。

### 准入控制（Admission Control）

准入控制是 Kubernetes API 服务器的一个关键安全和功能扩展机制。它发生在 API 请求经过认证和授权之后，但在对象持久化到 etcd 之前。准入控制器可以：

- **修改（Mutate）请求的对象：例如，自动给 Pod 添加一些 sidecar 容器，或者设置默认值。
    
- **验证（Validate）请求的对象：例如，检查 Pod 的镜像是否来自可信仓库，或者确保资源请求和限制符合策略。
    
- **拒绝**不符合策略的请求。

`admissionregistration.k8s.io` 组中包含的两个最主要的资源是：

- **`MutatingWebhookConfiguration` (v1)**：用于配置**修改型准入 Webhook**。当有特定的 API 请求（例如创建 Pod）到达时，API 服务器会向配置的 Webhook 发送请求，Webhook 可以修改这个请求的对象。
    
- **`ValidatingWebhookConfiguration` (v1)**：用于配置**验证型准入 Webhook**。类似地，API 服务器会向 Webhook 发送请求，Webhook 负责验证请求的对象是否符合规则，如果不符合则可以拒绝该请求。


# update Deployment
You have two options to Update() this Deployment:
1. Modify the "deployment" variable and call: Update(deployment). This works like the "kubectl replace" command and it overwrites/loses changes made by other clients between you Create() and Update() the object.

2. Modify the "result" returned by Get() and retry Update(result) until you no longer get a conflict error. This way, you can preserve changes made by other clients between Create() and Update(). This is implemented below using the retry utility package included with client-go. (RECOMMENDED)

![[Pasted image 20250627094636.png]]

## Concurrency Control and Consistency
### resource version
一个不透明值，表示此对象的内部版本，客户端可以使用它来确定对象何时发生了更改。可用于乐观并发、更改检测以及对单个或一组资源的监视操作。客户端必须将这些值视为不透明值，并将其原封不动地传回服务器。它们可能仅对特定资源或资源集有效。

Kubernetes leverages the concept of _resource versions_ to achieve optimistic concurrency. All Kubernetes resources have a "resourceVersion" field as part of their metadata. This resourceVersion is a string that identifies the internal version of an object that can be used by clients to determine when objects have changed. When a record is about to be updated, its version is checked against a pre-saved value, and if it doesn't match, the update fails with a StatusConflict (HTTP status code 409).

The resourceVersion is changed by the server every time an object is modified. If resourceVersion is included with the PUT operation the system will verify that there have not been other successful mutations to the resource during a read/modify/write cycle, by verifying that the current value of resourceVersion matches the specified value.

The resourceVersion is currently backed by [etcd's mod_revision](https://etcd.io/docs/latest/learning/api/#key-value-pair). However, it's important to note that the application should _not_ rely on the implementation details of the versioning system maintained by Kubernetes. We may change the implementation of resourceVersion in the future, such as to change it to a timestamp or per-object counter.

The only way for a client to know the expected value of resourceVersion is to have received it from the server in response to a prior operation, typically a GET. This value MUST be treated as opaque by clients and passed unmodified back to the server. Clients should not assume that the resource version has meaning across namespaces, different kinds of resources, or different servers. Currently, the value of resourceVersion is set to match etcd's sequencer. You could think of it as a logical clock the API server can use to order requests. However, we expect the implementation of resourceVersion to change in the future, such as in the case we shard the state by kind and/or namespace, or port to another storage system.

In the case of a conflict, the correct client action at this point is to GET the resource again, apply the changes afresh, and try submitting again. This mechanism can be used to prevent races like the following:

resourceVersion may be used as a precondition for other operations (e.g., GET, DELETE) in the future, such as for read-after-write consistency in the presence of caching.

"Watch" operations specify resourceVersion using a query parameter. It is used to specify the point at which to begin watching the specified resources. This may be used to ensure that no mutations are missed between a GET of a resource (or list of resources) and a subsequent Watch, even if the current version of the resource is more recent. This is currently the main reason that list operations (GET on a collection) return resourceVersion.

在 Kubernetes (K8s) 中，`resourceVersion` 是一个非常重要的概念，它主要用于**确保数据的一致性**和**实现乐观并发控制**。

---

### `resourceVersion` 的作用

你可以把 `resourceVersion` 理解为 K8s 对象（如 Pod、Deployment、Service 等）在 etcd 中存储的**版本号**或**快照ID**。每次 K8s 对象发生变化（创建、更新、删除），它的 `resourceVersion` 都会随之更新。

它的核心作用体现在以下几个方面：

1. **乐观并发控制 (Optimistic Concurrency Control)**
    
    - 当客户端（例如 `kubectl`、控制器或其他 K8s 服务）想要更新一个 K8s 对象时，它需要先获取该对象的当前版本，其中包含了当前的 `resourceVersion`。
        
    - 在提交更新请求时，客户端会将获取到的 `resourceVersion` 也一起发送给 K8s API Server。
        
    - API Server 会比较客户端提交的 `resourceVersion` 和 etcd 中存储的当前对象的 `resourceVersion`。
        
        - **如果两者匹配**：说明在客户端获取对象到提交更新期间，该对象没有被其他方修改过，API Server 会接受并处理这个更新请求，然后更新对象的 `resourceVersion`。
            
        - **如果两者不匹配**：说明在该期间对象已经被其他方修改过了，API Server 会拒绝这个更新请求，并返回一个冲突错误（通常是 `409 Conflict`）。客户端需要重新获取最新版本，然后再次尝试更新。
            
    - 这种机制避免了“**写丢失**”的问题，确保了多个并发修改操作的正确性。
        
2. **数据一致性**
    
    - `resourceVersion` 用于在获取（`GET`）、列出（`LIST`）和监视（`WATCH`）资源时表达数据一致性要求。
        
    - **GET/LIST 请求**：客户端可以指定 `resourceVersion` 来请求特定版本的数据，或者确保获取到最新的数据。
        
    - **WATCH 机制**：这是 `resourceVersion` 最重要的应用场景之一。
        
        - 当客户端启动一个 `WATCH` 请求来监听某个资源的变化时，它通常会指定一个 `resourceVersion`。
            
        - API Server 会从这个指定的 `resourceVersion` 之后的所有事件开始推送给客户端。
            
        - 这确保了客户端不会丢失任何在它开始监听之后发生的事件，即使在监听开始之前资源已经发生了多次变化。
            
        - 如果客户端的 `resourceVersion` 太旧，或者 API Server 无法从指定的版本开始提供事件（例如，因为历史数据已经被清理），API Server 可能会要求客户端重新从头开始列表并监听。
            

---

### `resourceVersion` 的特性

- `resourceVersion` 是一个**字符串**，不应假设它是数字或可排序的。你只能比较两个 `resourceVersion` 是否相等。
    
- 它对客户端是**不透明的**，客户端不应该尝试解析或修改它，而应该将其视为一个标识符，在需要时原样传回服务器。
    
- 不同的资源类型，甚至不同的 K8s 集群，其 `resourceVersion` 的生成方式和表现可能有所不同。

