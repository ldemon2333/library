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

## Admission control extension points
Within the full [list](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#what-does-each-admission-controller-do), there are three special controllers: [MutatingAdmissionWebhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook), [ValidatingAdmissionWebhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#validatingadmissionwebhook), and [ValidatingAdmissionPolicy](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#validatingadmissionpolicy). The two webhook controllers execute the mutating and validating (respectively) [admission control webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#admission-webhooks) which are configured in the API. ValidatingAdmissionPolicy provides a way to embed declarative validation code within the API, without relying on any external HTTP callouts.

You can use these three admission controllers to customize cluster behavior at admission time.

Okay, let's break down these three important Kubernetes admission controllers: **MutatingAdmissionWebhook**, **ValidatingAdmissionWebhook**, and **ValidatingAdmissionPolicy**.

### The Three Special Controllers

Now, let's look at the three you mentioned:

#### 1. MutatingAdmissionWebhook

- **Purpose:** This controller's job is to **mutate** or change an incoming API request. Think of it as a pre-processing step where you can inject, modify, or default values on Kubernetes objects before they are saved.
- **How it Works:** Instead of having its own built-in logic, `MutatingAdmissionWebhook` acts as a "gatekeeper" that sends the incoming API request to an **external service** (your webhook server) over HTTP. Your webhook server then processes the request, makes any necessary changes to the Kubernetes object, and sends it back to the API server.
- **Use Cases:**
    - **Automatic sidecar injection:** Automatically add a `Istio` proxy or a logging agent container to new pods.
    - **Setting default values:** Ensure certain labels, annotations, or resource requests are present on objects if they're not specified by the user.
    - **Adding security contexts:** Automatically apply specific `seccomp` profiles or `AppArmor` annotations to pods.
- **Configuration:** You define a `MutatingWebhookConfiguration` object in Kubernetes that tells the API server:
    - Which operations (e.g., `CREATE`, `UPDATE`) to intercept.
    - Which resources (e.g., `pods`, `deployments`) to intercept.
    - The URL of your external webhook service.

#### 2. ValidatingAdmissionWebhook

- **Purpose:** This controller's job is to **validate** an incoming API request. It checks if the object conforms to specific rules or policies. If the object doesn't meet the criteria, the request is rejected, and the object is _not_ persisted.
- **How it Works:** Similar to its mutating counterpart, `ValidatingAdmissionWebhook` sends the incoming API request to an **external service** (your webhook server). Your webhook server then evaluates the request against your defined rules. If the validation fails, the webhook server returns an error, and the Kubernetes API server rejects the request. If it passes, the request continues.
- **Use Cases:**
    - **Enforcing security policies:** Prevent privileged containers, ensure specific image registries are used, or disallow certain `Capabilities`.
    - **Resource limits:** Ensure all pods have resource limits defined.
    - **Naming conventions:** Enforce specific naming patterns for namespaces or resources.
    - **Prohibiting certain actions:** Prevent deletion of critical namespaces or resources.
- **Configuration:** You define a `ValidatingWebhookConfiguration` object, similar to mutating webhooks, specifying what to intercept and the URL of your external webhook service.

#### 3. ValidatingAdmissionPolicy

- **Purpose:** This is a newer, more declarative way to perform **validation** directly within the Kubernetes API server, **without relying on external HTTP callouts to a separate webhook server**. It allows you to embed your validation logic using the Common Expression Language (CEL).
- **How it Works:** Instead of sending the request to an external service, `ValidatingAdmissionPolicy` evaluates rules defined directly within the Kubernetes API object itself using **CEL expressions**. If the CEL expression evaluates to `false` or an error, the request is rejected.
- **Use Cases:**
    - **Simple policy enforcement:** Where the logic is straightforward and doesn't require complex external business logic or data lookups.
    - **Reducing operational overhead:** You don't need to deploy, manage, and scale a separate webhook server.
    - **Faster validation:** As it's evaluated directly by the API server, it can be slightly faster than an external webhook call.
- **Key Differences from Validating Webhooks:**
    - **No external service:** This is the biggest difference. No need to run your own server.
    - **Declarative:** Policies are defined directly in YAML using CEL, making them version-controlled and auditable within your cluster.
    - **Validation only:** Currently, `ValidatingAdmissionPolicy` is only for validation; it cannot mutate objects.
- **Configuration:** You define `ValidatingAdmissionPolicy` and `ValidatingAdmissionPolicyBinding` objects in Kubernetes. The policy defines the rules, and the binding links those rules to specific resources and operations.

---

### Key Takeaways and When to Use Which

- **Webhooks (`MutatingAdmissionWebhook`, `ValidatingAdmissionWebhook`):**
    
    - **Pros:** Highly flexible, can implement complex logic, integrate with external systems, perform both mutation and validation.
    - **Cons:** Requires deploying and managing an external service, introduces an external dependency, can add latency.
    - **When to use:** When you need to mutate objects, when your validation logic is complex, requires external data, or when you need to integrate with other services (e.g., an IP address management system).
- **ValidatingAdmissionPolicy:**
    
    - **Pros:** Native Kubernetes feature, no external service required, simpler to manage for basic validation, potentially lower latency.
    - **Cons:** Limited to validation (no mutation), logic must be expressible in CEL (can be less powerful than arbitrary code), newer feature (may have fewer advanced examples/community support compared to webhooks).
    - **When to use:** For straightforward validation rules where you want to avoid the overhead of managing a separate webhook server. It's an excellent choice for common policy enforcement directly within Kubernetes.

In essence, webhooks offer maximum flexibility at the cost of operational complexity, while `ValidatingAdmissionPolicy` provides a simpler, native solution for validation, especially for common policy use cases.

Does this explanation help clarify the roles of these three controllers for you?

## Admission control phases
Two phases. 

If any of the controllers in either phase reject the request, the entire request is rejected immediately and an error is returned to the end-user.

Finally, in addition to sometimes mutating the object in question, admission controllers may sometimes have side effects, that is, mutate related resources as part of request processing. Incrementing quota usage is the canonical example of why this is necessary. Any such side-effect needs a corresponding reclamation or reconciliation process, as a given admission controller does not know for sure that a given request will pass all of the other admission controllers.

The ordering of these calls can be seen below.

![Sequence diagram for kube-apiserver handling requests during the admission phase showing mutation webhooks, followed by validatingadmissionpolicies and finally validating webhooks. It shows that the continue until the first rejection, or being accepted by all of them. It also shows that mutations by mutating webhooks cause all previously called webhooks to be called again.](https://kubernetes.io/docs/reference/access-authn-authz/admission-control-phases.svg)

# What does each admission controller do?
- AlwaysPullImages


# 示例
理解了！你想看一些 **Admission Control** 的具体使用示例，这样能更好地明白它们在 Kubernetes 中是如何工作的。

Admission Controllers 在 Kubernetes 中非常强大，可以用来强制执行各种集群范围的策略和自动化任务。下面我将为你展示几个常见的示例，涵盖了 **Mutating Admission Webhook**、**Validating Admission Webhook** 和 **ValidatingAdmissionPolicy** 的应用。

---

### 1. 强制 Pod 必须设置资源限制 (Validating Admission Webhook)

这是一个非常常见的需求：确保集群中的所有 Pod 都定义了 CPU 和内存的请求（requests）和限制（limits），以避免资源争抢和保证服务质量。

**场景:** 用户尝试部署一个没有设置资源限制的 Pod，Admission Controller 应该拒绝这个请求。

**实现方式 (Validating Admission Webhook):**

1. 编写一个 Webhook Server (例如，使用 Go 语言):
    
    这个服务会监听 /validate-pod-resources 这样的 HTTP 端点。当 Kubernetes API Server 将 Pod 创建请求发送过来时，它会检查 Pod 的 Spec 中是否有 resources.requests 和 resources.limits。如果没有，就返回一个拒绝的响应。
    
```
  // 简化的 Go 代码片段
func validatePodResources(ar v1beta.AdmissionReview) *v1beta.AdmissionResponse {
    pod := corev1.Pod{}
    json.Unmarshal(ar.Request.Object.Raw, &pod)

    if pod.Spec.Containers == nil {
        return &v1beta.AdmissionResponse{Allowed: true}
    }

    for _, container := range pod.Spec.Containers {
        if container.Resources.Requests == nil || container.Resources.Limits == nil {
            return &v1beta.AdmissionResponse{
                Allowed: false,
                Result:  &metav1.Status{Message: "All containers must define resource requests and limits."},
            }
        }
    }
    return &v1beta.AdmissionResponse{Allowed: true}
}
```

1. 将 Webhook Server 部署到 Kubernetes 集群中:
    
    这通常是一个 Deployment 和 Service，Service 会暴露一个 HTTPS 端点。
    
2. 创建 ValidatingWebhookConfiguration 对象:
    
    这个 Kubernetes 对象告诉 API Server 何时调用你的 Webhook。
    
    YAML
    
    ```
    apiVersion: admissionregistration.k8s.io/v1
    kind: ValidatingWebhookConfiguration
    metadata:
      name: enforce-resource-limits
    webhooks:
      - name: validate-pod-resources.example.com
        rules:
          - operations: ["CREATE"] # 只在创建 Pod 时触发
            apiGroups: [""]
            apiVersions: ["v1"]
            resources: ["pods"]
        clientConfig:
          service:
            name: my-webhook-service # 你的 Webhook Service 名称
            namespace: default # 你的 Webhook Service 所在命名空间
            path: "/validate-pod-resources" # Webhook Server 的路径
          caBundle: <Your_Service_CA_Cert_Base64_Encoded> # 用于验证 Webhook Server 证书的 CA 证书
        sideEffects: None
        admissionReviewVersions: ["v1", "v1beta1"]
        failurePolicy: Fail # 失败时拒绝请求
    ```
    

**效果:** 任何尝试创建没有资源限制的 Pod 的请求都会被这个 Webhook 拦截并拒绝。

---

### 2. 自动注入 Istio Sidecar (Mutating Admission Webhook)

这是 **Mutating Admission Webhook** 的经典应用，尤其在服务网格 (Service Mesh) 中非常常见。

**场景:** 当用户部署一个 Pod 时，你希望自动为这个 Pod 注入一个 Istio Sidecar 容器，而不需要用户手动修改其 Deployment 配置。

**实现方式 (Mutating Admission Webhook):**

1. **Webhook Server:** Istio 本身就提供了一个 `sidecar-injector` 的 Webhook Server。当一个 Pod 请求到达时，这个 Webhook 会检查 Pod 的标签（例如 `istio-injection: enabled`）。如果标签存在，它就会修改 Pod 的 Spec，添加 Istio Sidecar 容器、Volume、Init 容器等必要配置。
    
2. **MutatingWebhookConfiguration 对象:** Istio 安装时会部署一个类似的配置：
    
    YAML
    
    ```
    apiVersion: admissionregistration.k8s.io/v1
    kind: MutatingWebhookConfiguration
    metadata:
      name: istio-sidecar-injector
    webhooks:
      - name: sidecar-injector.istio.io
        clientConfig:
          service:
            name: istiod
            namespace: istio-system
            path: "/inject"
          caBundle: <Istio_CA_Cert_Base64_Encoded>
        rules:
          - operations: [ "CREATE" ]
            apiGroups: [""]
            apiVersions: ["v1"]
            resources: ["pods"]
            scope: "Namespaced" # 作用于命名空间级别
        namespaceSelector:
          matchLabels:
            istio-injection: enabled # 只有带这个标签的命名空间才触发注入
        sideEffects: None
        admissionReviewVersions: ["v1", "v1beta1"]
        # ... 其他配置
    ```
    

**效果:** 只要你在命名空间上打上 `istio-injection: enabled` 标签，那么在这个命名空间中创建的任何 Pod 都会自动被 Istio 的 Sidecar 注入，无需用户干预。

---

### 3. 使用 ValidatingAdmissionPolicy 强制镜像仓库前缀

这是 **ValidatingAdmissionPolicy** 的一个很好的用例，因为它是一个相对简单但关键的验证，不需要外部服务。

**场景:** 你希望强制所有 Pod 只能从你公司的私有镜像仓库拉取镜像，例如 `my-private-registry.com/`。

**实现方式 (ValidatingAdmissionPolicy):**

1. **定义 ValidatingAdmissionPolicy:**
    
    YAML
    
    ```
    apiVersion: admissionregistration.k8s.io/v1alpha1 # 注意这是 v1alpha1 版本
    kind: ValidatingAdmissionPolicy
    metadata:
      name: enforce-image-registry
    spec:
      failurePolicy: Fail # 如果验证失败，拒绝请求
      matchConstraints:
        resourceRules:
        - apiGroups: [""]
          apiVersions: ["v1"]
          operations: ["CREATE", "UPDATE"]
          resources: ["pods"]
      validations:
        - expression: "object.spec.containers.all(c, c.image.startsWith('my-private-registry.com/'))"
          message: "All container images must come from 'my-private-registry.com/'."
    ```
    
    - `expression`: 使用 **CEL (Common Expression Language)** 来定义验证逻辑。这里的意思是“所有容器的镜像（`c.image`）都必须以 `my-private-registry.com/` 开头”。
    - `message`: 如果验证失败，返回给用户的错误信息。
2. 定义 ValidatingAdmissionPolicyBinding:
    
    这个绑定对象将策略应用到特定的作用域（例如，集群中的所有 Pod）。
    
    YAML
    
    ```
    apiVersion: admissionregistration.k8s.io/v1alpha1
    kind: ValidatingAdmissionPolicyBinding
    metadata:
      name: enforce-image-registry-binding
    spec:
      policyName: enforce-image-registry # 引用上面定义的策略
      matchResources:
        namespaceSelector:
          matchLabels: {} # 空标签选择器意味着匹配所有命名空间
        # 可以根据需要添加其他规则，例如只匹配特定命名空间
    ```
    

**效果:** 任何尝试创建或更新 Pod，并且其中包含不以 `my-private-registry.com/` 开头的镜像的请求，都会被 API Server 直接拒绝，并显示你定义的错误消息。
