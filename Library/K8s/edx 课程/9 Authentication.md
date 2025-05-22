每个到达 API 服务器的 API 请求都必须经过多个控制阶段，才能被服务器接受并执行。在本章中，我们将学习 Kubernetes API 访问控制的身份验证、授权和准入控制阶段。

By the end of this chapter, you should be able to:

- Discuss the authentication, authorization, and access control stages of the Kubernetes API access.
- Understand the different kinds of Kubernetes users.
- Explore the different modules for authentication and authorization.

在 Kubernetes 或更广义的软件架构中，**API endpoint** 是指客户端可以通过网络访问的、提供特定服务的 URL 路径。

---

## 📌 API Endpoint 是什么？

**API endpoint** 就是 **API 提供的一个具体访问地址**，用于客户端与服务器之间的通信。

在形式上，它通常包括：

```
<协议>://<主机>:<端口>/<路径>
```

例如：

```
https://my-api-server.com/api/v1/namespaces/default/pods
```

---

## 🔍 在 Kubernetes 中的 API Endpoint

Kubernetes 提供了一个强大的 REST API，每个功能（比如获取 Pod、创建 Deployment、获取日志）都有自己的 endpoint。例如：

|功能|API Endpoint 示例|
|---|---|
|列出所有 Pods|`/api/v1/pods`|
|获取某个 Pod|`/api/v1/namespaces/default/pods/<pod-name>`|
|创建一个 Deployment|`/apis/apps/v1/namespaces/default/deployments`|
|获取集群信息|`/version`|

完整地址是基于 Kubernetes API Server 的地址，比如：

```
https://<k8s-api-server>:6443/api/v1/namespaces/default/pods
```

---

## ✅ 例子说明

假如你有一个程序需要访问 Kubernetes API 来获取某个 namespace 下的 Pods，它需要访问：

```
GET https://<k8s-api-server>/api/v1/namespaces/default/pods
```

这个 URL 就是一个 **API endpoint**，它指定了访问什么资源、在哪个 namespace 下、以及用什么操作（GET、POST 等）。

---

## 🚀 小结

|项目|说明|
|---|---|
|API endpoint|是一个特定的 URL，用于访问某个 API 功能|
|在 K8s 中|用于访问资源（Pods、Nodes、Deployments 等）|
|使用方式|通常是通过 HTTP 请求访问（curl、客户端库等）|

To access and manage K8s resources or objects in the cluster, we need to access a specific API endpoint on the API server.
- Authentication
- Authorization
- Admission Control

- 身份验证
根据 API 请求中提供的凭证对用户进行身份验证。
- 授权
授权已通过身份验证的用户提交的 API 请求。
- 准入控制
验证和/或修改用户请求的软件模块。

![[Pasted image 20250513121531.png]]

# Authentication
Kubernetes 没有名为“用户”的对象，也不在其对象存储中存储用户名或其他相关详细信息。然而，即使没有这些，Kubernetes 也可以使用用户名进行 API 访问控制的身份验证阶段，以及请求日志记录。

k8s supports two kind of users:
- Normal Users
它们通过独立服务（如用户/客户端证书、列出用户名/密码的文件、Google 帐户等）在 Kubernetes 集群外部进行管理。
- Service Accounts
服务账户允许集群内的进程与 API 服务器通信以执行各种操作。大多数服务账户是通过 API 服务器自动创建的，但也可以手动创建。服务账户绑定到特定的命名空间，并将相应的凭据以 Secret 的形式挂载到 API 服务器进行通信。

如果配置正确，Kubernetes 还可以支持匿名请求，以及来自普通用户和服务账户的请求。它还支持用户模拟，允许一个用户扮演另一个用户，这对于管理员在排除授权策略故障时非常有用。

Kubernetes 使用一系列身份验证模块进行身份验证：

- X509 客户端证书
要启用客户端证书身份验证，我们需要通过将 --client-ca-file=SOMEFILE 选项传递给 API 服务器来引用包含一个或多个证书颁发机构的文件。文件中提到的证书颁发机构将验证用户提交给 API 服务器的客户端证书。本章末尾提供了一个演示视频，介绍此主题。
- 静态令牌文件
我们可以使用 --token-auth-file=SOMEFILE 选项将包含预定义承载令牌的文件传递给 API 服务器。目前，这些令牌将无限期有效，并且必须在重启 API 服务器后才能更改。
- 引导令牌
用于引导新的 Kubernetes 集群的令牌。
- 服务账户令牌
自动启用的身份验证器，使用签名的承载令牌来验证请求。这些令牌通过服务账户准入控制器附加到 Pod，从而允许集群内的进程与 API 服务器通信。
- OpenID Connect 令牌
OpenID Connect 帮助我们连接 OAuth2 提供商，例如 Microsoft Entra ID（以前称为 Azure Active Directory）、Salesforce 和 Google，从而将身份验证任务转移至外部服务。
- Webhook 令牌身份验证
使用基于 Webhook 的身份验证，可以将持有者令牌的验证任务转移至远程服务。
- 身份验证代理
允许编写额外的身份验证逻辑。

我们可以启用多个身份验证器，第一个成功验证请求的模块将缩短验证过程。为了确保用户身份验证成功，我们应该至少启用两种方法：服务帐户令牌身份验证器和其中一种用户身份验证器。

# Authorization
身份验证成功后，用户可以发送 API 请求来执行不同的操作。Kubernetes 使用各种授权模块对这些 API 请求进行授权，允许或拒绝请求。

Kubernetes 会审核 API 请求的一些属性包括用户、组、资源、命名空间或 API 组等。接下来，Kubernetes 会根据策略评估这些属性。如果评估成功，则允许请求，否则拒绝请求。与身份验证步骤类似，授权也包含多个模块或授权器。一个 Kubernetes 集群可以配置多个模块，并按顺序检查每个模块。如果任何授权器批准或拒绝了请求，则会立即返回该决策。

## Node
节点授权是一种特殊用途的授权模式，专门授权 kubelet 发出的 API 请求。它授权 kubelet 对服务、端点或节点的读取操作，以及对节点、pod 和事件的写入操作。

## Attrubute-Based Access Control (ABAC)
借助 ABAC 授权器，Kubernetes 可以授予 API 请求访问权限，这些请求将策略与属性相结合。在以下示例中，用户 bob 只能读取命名空间 lfs158 中的 Pod。

## Webhook
在 Webhook 模式下，Kubernetes 可以请求第三方服务做出授权决策，授权成功返回 true，失败返回 false。为了启用 Webhook 授权器，我们需要使用 --authorization-webhook-config-file=SOME_FILENAME 选项启动 API 服务器，其中 SOME_FILENAME 是远程授权服务的配置。更多详情，请参阅 Webhook 模式文档。

## Role-Based Access Control (RBAC)
