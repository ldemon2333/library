很好，我们来深入探讨 **动态准入控制器 (Dynamic Admission Control)** 和 **静态准入控制器 (Static Admission Control)** 之间的区别。

---

### Kubernetes 准入控制器概述

在深入动态和静态区别之前，先回顾一下准入控制器在 Kubernetes 中的位置：

当一个 API 请求（例如，创建 Pod、修改 Deployment）到达 Kubernetes **API Server** 后，它会经历以下阶段：

1. **认证 (Authentication)**：验证请求者的身份。
2. **授权 (Authorization)**：检查请求者是否有权限执行该操作。
3. **准入控制 (Admission Control)**：在此阶段，请求会被一系列准入控制器拦截和处理。控制器可以：
    - **改变 (Mutate)** 请求中的对象。
    - **验证 (Validate)** 请求中的对象，如果违反策略则拒绝。
4. **持久化到 etcd**：如果通过了所有准入控制器，对象最终会被保存到 etcd (Kubernetes 的数据存储)。

---

### 静态准入控制器 (Static Admission Controllers)

**特点：**

- **内置于 kube-apiserver:** 静态准入控制器是 Kubernetes API Server **内置的代码模块**。它们是 `kube-apiserver` 二进制文件的一部分，随着 API Server 一起编译和运行。
- **配置方式:** 它们通常通过修改 `kube-apiserver` 的启动参数（例如 `--enable-admission-plugins` 和 `--admission-control-config-file`）来启用或禁用，并配置它们的行为。
- **固定功能:** 每个静态控制器都有其预定义的功能和逻辑，比如 `NamespaceLifecycle` 确保不会在正在删除的命名空间中创建新对象，`LimitRanger` 应用命名空间级别的资源限制，`ServiceAccount` 自动化 ServiceAccount 相关的任务等。
- **升级耦合:** 如果要更新或修改这些控制器的逻辑，需要升级或重新部署 `kube-apiserver` 组件。
- **开箱即用:** 许多重要的 Kubernetes 功能都依赖于这些内置的静态控制器来正常工作。

**示例:**

- **`NamespaceLifecycle`**: 当一个命名空间被标记为删除时，阻止在该命名空间内创建任何新对象。
- **`LimitRanger`**: 对命名空间内的 Pod、Container 等强制执行默认的资源限制（CPU/内存请求和限制）。
- **`ServiceAccount`**: 自动为 Pod 注入 `ServiceAccount` Token。
- **`ResourceQuota`**: 强制执行命名空间级别的资源配额。
- **`AlwaysPullImages`**: 强制所有 Pod 的镜像拉取策略为 `Always`，确保每次都从镜像仓库拉取最新镜像。

---

### 动态准入控制器 (Dynamic Admission Controllers)

**特点：**

- **外部服务实现:** 动态准入控制器不是内置在 `kube-apiserver` 中，而是作为**独立的外部服务**运行在集群中（通常是作为 Pod 部署的 Webhook 服务）。
- **运行时注册:** 它们通过 Kubernetes API 对象（`MutatingWebhookConfiguration` 和 `ValidatingWebhookConfiguration`）在**运行时注册**到 API Server。这意味着你可以在不重启或重新配置 API Server 的情况下，动态地添加、更新或删除这些控制器。
- **高度可定制:** 这是它们最大的优势。你可以编写任意复杂的逻辑，使用任何你喜欢的编程语言（只要能处理 HTTP 请求），从而实现非常具体的业务需求和安全策略。
- **基于 Webhook:** API Server 通过 **HTTP 回调 (webhook)** 的方式与这些外部服务通信，将请求发送给它们进行处理。
- **分离和解耦:** 动态控制器与 API Server 分离，使得策略的开发、部署和管理更加灵活。

**主要类型:**

- **`MutatingAdmissionWebhook`**: 负责将请求发送给**变异 (mutating)** Webhook 服务，该服务可以修改传入的 Kubernetes 对象。
- **`ValidatingAdmissionWebhook`**: 负责将请求发送给**验证 (validating)** Webhook 服务，该服务可以检查传入的 Kubernetes 对象并根据策略决定是否允许该请求。
- **`ValidatingAdmissionPolicy`** (有时也被归类为一种特殊的动态控制器，因为它也是在运行时配置，但它的验证逻辑是声明式的，直接嵌入在 API 中，无需外部 HTTP 服务): 提供了一种在 Kubernetes API 中直接定义声明性验证逻辑的方法，使用 **CEL (Common Expression Language)**。它不需要独立的 Webhook 服务。

**示例:**

- **Istio Sidecar 注入**: 自动为 Pod 注入 Istio 代理容器。
- **Kyverno / OPA Gatekeeper**: 强大的策略引擎，可以实现非常细粒度的策略，例如：
    - 强制所有 Pod 镜像只能来自特定的私有仓库。
    - 阻止特权容器的创建。
    - 要求所有 Deployment 必须有 Owner 标签。
    - 自动为 Ingress 添加 TLS 配置。
- **自定义默认值**: 自动为新创建的 Namespace 添加特定标签或注解。

---

### 主要区别总结

|   |   |   |
|---|---|---|
|**特点**|**静态准入控制器 (Static Admission Controllers)**|**动态准入控制器 (Dynamic Admission Controllers)**|
|**位置**|内置于 `kube-apiserver` 二进制文件中|作为独立的外部服务运行 (通常是 Webhook 服务)|
|**配置方式**|`kube-apiserver` 启动参数 (`--enable-admission-plugins`) 或配置文件|Kubernetes API 对象 (`MutatingWebhookConfiguration`, `ValidatingWebhookConfiguration`, `ValidatingAdmissionPolicy` 等)|
|**功能**|预定义、固定的逻辑|高度可定制，可实现任意复杂逻辑|
|**升级/修改**|需要重启或升级 `kube-apiserver`|独立部署和升级，不影响 `kube-apiserver`|
|**灵活性**|有限，适用于通用且强制的集群行为|极高，适用于特定业务需求、安全策略和自动化|
|**依赖**|无外部服务依赖|依赖外部 Webhook 服务 (除了 `ValidatingAdmissionPolicy`)|
|**新特性**|核心 Kubernetes 特性通常依赖静态控制器|扩展 Kubernetes 功能，实现自定义策略|

---

### 什么时候使用哪种？

- **使用静态准入控制器：** 当你想要启用或配置 Kubernetes **内置的、核心的、强制性**的集群行为时。例如，你总是会启用 `LimitRanger` 和 `ResourceQuota` 来管理资源。这些是平台级别的基础设施策略。
- **使用动态准入控制器：** 当你：
    - 需要实现**自定义的业务逻辑或安全策略**，而这些策略是 Kubernetes 内置控制器无法满足的。
    - 希望**自动化**某些任务（例如 sidecar 注入、默认值设置）。
    - 需要**更细粒度地控制**资源对象的创建、修改、删除。
    - 希望将策略逻辑与核心的 Kubernetes 组件**解耦**，以便独立开发和部署。
    - 尤其对于 `ValidatingAdmissionPolicy`，它适用于**声明式且不依赖外部服务**的验证场景，比如简单的字段格式检查或强制命名约定。

理解这两种类型的区别，能够帮助你更好地设计和管理你的 Kubernetes 集群的策略和自动化流程。