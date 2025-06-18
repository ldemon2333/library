An admission controller is a piece of code that intercepts requests to the Kubernetes API server prior to persistence of the resource, but after the request is authenticated and authorized.

准入控制器是一段代码，它在资源持久化之前、但在请求经过身份验证和授权之后拦截对 Kubernetes API 服务器的请求。

# What are they?
Admission controllers apply to requests that create, delete, or modify objects. Admission controllers can also block custom verbs, such as a request to connect to a pod via an API server proxy. Admission controllers do _not_ (and cannot) block requests to read (**get**, **watch** or **list**) objects, because reads bypass the admission control layer.

在 Kubernetes 中，"custom verbs"（自定义动词）是指那些不直接对应于标准 RESTful API 操作（如 GET、POST、PUT、DELETE）的动词，而是由 Kubernetes API 或扩展的 API 定义的特殊操作。

让我们来详细解释一下：

1. **标准 RESTful 动词**:
    
    - `GET`：用于读取（获取、查看）资源。
    - `POST`：用于创建新资源。
    - `PUT`：用于完全替换现有资源（更新）。
    - `DELETE`：用于删除资源。
    - `PATCH`：用于部分更新现有资源。
2. Kubernetes 中的 "动词":
    
    Kubernetes 的 API 是基于 RESTful 原则构建的，所以它也使用上述标准 HTTP 动词来表示对资源的操作。例如：
    
    - 当你 `kubectl create` 一个 Pod 时，Kubernetes API Server 接收到的请求是一个 `POST` 请求。
    - 当你 `kubectl get pods` 时，它会发送一个 `GET` 请求。
    - 当你 `kubectl delete pod my-pod` 时，它会发送一个 `DELETE` 请求。
3. 什么是 "Custom Verbs" (自定义动词):
    
    除了这些标准的 CRUD（创建、读取、更新、删除）操作之外，Kubernetes 允许定义一些特殊的、非 CRUD 的操作，这些就被称为 "custom verbs"。这些自定义动词通常用于表示对资源进行更复杂的、特定于业务逻辑的操作，而不是简单的创建、修改或删除。
    
    例如，文档中提到的 "a request to connect to a pod via an API server proxy" 就是一个典型的自定义动词的例子。当你运行 `kubectl exec` 命令连接到一个 Pod 时，这个操作不是一个简单的 `GET`、`POST` 或 `DELETE`。它是一个特殊的“连接”操作，通过 API Server 代理与 Pod 建立连接。Kubernetes API 将此识别为 `CONNECT` 动词。
    
    在 RBAC（基于角色的访问控制）中，你可以为用户或服务帐户授予对特定资源执行这些自定义动词的权限。例如，你可以定义一个 RBAC 规则，允许某个用户对 Pod 执行 `connect` 动词，从而允许他们使用 `kubectl exec` 进入 Pod。
    
4. Admission Controllers 和 Custom Verbs 的关系:
    
    Admission Controllers 在请求被认证和授权之后，但在对象持久化到 etcd 之前拦截请求。它们可以：
    
    - **阻止** `create`、`delete` 或 `modify`（包括 `update` 和 `patch`）对象的请求。
    - **阻止** `custom verbs` 的请求，例如上面提到的 `connect` 到 Pod。
    
    **重要的一点是：Admission Controllers 无法阻止 `read`（`get`、`watch` 或 `list`）请求。** 这是因为读取操作在 Kubernetes API Server 的处理流程中，比 Admission Control 层的执行更早。所以，如果你想限制用户对某个资源的读取权限，你需要通过 RBAC 来实现，而不是 Admission Controllers。
    

总结来说，"custom verbs" 是 Kubernetes API 中定义的、超越标准 CRUD 操作的特定行为，允许对资源执行更高级或特殊的操作，并且这些操作可以被 Admission Controllers 拦截和控制。

Admission control mechanisms may be _validating_, _mutating_, or both. Mutating controllers may modify the data for the resource being modified; validating controllers may not.

