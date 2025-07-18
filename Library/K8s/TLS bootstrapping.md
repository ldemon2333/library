在 Kubernetes 集群中，工作节点上的组件（kubelet 和 kube-proxy）需要与 Kubernetes 控制平面组件（特别是 kube-apiserver）通信。为了确保通信的私密性、不受干扰，并确保集群中的每个组件都与另一个可信组件通信，我们强烈建议在节点上使用客户端 TLS 证书。

引导这些组件（尤其是需要证书才能与 kube-apiserver 安全通信的工作节点）的正常过程可能非常具有挑战性，因为它通常超出了 Kubernetes 的职责范围，并且需要大量额外的工作。这反过来又会使集群的初始化或扩展变得非常困难。

In order to simplify the process, beginning in version 1.4, Kubernetes introduced a certificate request and signing API. The proposal can be found [here](https://github.com/kubernetes/kubernetes/pull/20439).

This document describes the process of node initialization, how to set up TLS client certificate bootstrapping for kubelets, and how it works.

# Initialization process
When a worker node starts up, the kubelet does the following:

1. Look for its `kubeconfig` file
2. Retrieve the URL of the API server and credentials, normally a TLS key and signed certificate from the `kubeconfig` file
3. Attempt to communicate with the API server using the credentials.

Assuming that the kube-apiserver successfully validates the kubelet's credentials, it will treat the kubelet as a valid node, and begin to assign pods to it.

Note that the above process depends upon:

- Existence of a key and certificate on the local host in the `kubeconfig`
- The certificate having been signed by a Certificate Authority (CA) trusted by the kube-apiserver

All of the following are responsibilities of whoever sets up and manages the cluster:

1. Creating the CA key and certificate
2. Distributing the CA certificate to the control plane nodes, where kube-apiserver is running
3. Creating a key and certificate for each kubelet; strongly recommended to have a unique one, with a unique CN, for each kubelet
4. Signing the kubelet certificate using the CA key
5. Distributing the kubelet key and signed certificate to the specific node on which the kubelet is running

The TLS Bootstrapping described in this document is intended to simplify, and partially or even completely automate, steps 3 onwards, as these are the most common when initializing or scaling a cluster.

## Bootstrap initialization
In the bootstrap initialization process, the following occurs:

1. kubelet begins
2. kubelet sees that it does _not_ have a `kubeconfig` file
3. kubelet searches for and finds a `bootstrap-kubeconfig` file
4. kubelet reads its bootstrap file, retrieving the URL of the API server and a limited usage "token"
5. kubelet connects to the API server, authenticates using the token
6. kubelet now has limited credentials to create and retrieve a certificate signing request (CSR)
7. kubelet creates a CSR for itself with the signerName set to `kubernetes.io/kube-apiserver-client-kubelet`
8. CSR is approved in one of two ways:
    - If configured, kube-controller-manager automatically approves the CSR
    - If configured, an outside process, possibly a person, approves the CSR using the Kubernetes API or via `kubectl`
9. Certificate is created for the kubelet
10. Certificate is issued to the kubelet
11. kubelet retrieves the certificate
12. kubelet creates a proper `kubeconfig` with the key and signed certificate
13. kubelet begins normal operation
14. Optional: if configured, kubelet automatically requests renewal of the certificate when it is close to expiry
15. The renewed certificate is approved and issued, either automatically or manually, depending on configuration.

The rest of this document describes the necessary steps to configure TLS Bootstrapping, and its limitations.

# Configuration
To configure for TLS bootstrapping and optional automatic approval, you must configure options on the following components:

- kube-apiserver
- kube-controller-manager
- kubelet
- in-cluster resources: `ClusterRoleBinding` and potentially `ClusterRole`

In addition, you need your Kubernetes Certificate Authority (CA).

# Certificate Authority
As without bootstrapping, you will need a Certificate Authority (CA) key and certificate. As without bootstrapping, these will be used to sign the kubelet certificate. As before, it is your responsibility to distribute them to control plane nodes.

For the purposes of this document, we will assume these have been distributed to control plane nodes at `/var/lib/kubernetes/ca.pem` (certificate) and `/var/lib/kubernetes/ca-key.pem` (key). We will refer to these as "Kubernetes CA certificate and key".

All Kubernetes components that use these certificates - kubelet, kube-apiserver, kube-controller-manager - assume the key and certificate to be PEM-encoded.

# kube-apiserver configuration
The kube-apiserver has several requirements to enable TLS bootstrapping:
- Recognizing CA that signs the client certificate
- Authenticating the bootstrapping kubelet to the `system:bootstrappers` group
- Authorize the bootstrapping kubelet to create a certificate signing request (CSR)

## Recongnizing client certificates


## Initial bootstrap authentication
In order for the bootstrapping kubelet to connect to kube-apiserver and request a certificate, it must first authenticate to the server. You can use any [authenticator](https://kubernetes.io/docs/reference/access-authn-authz/authentication/) that can authenticate the kubelet.

While any authentication strategy can be used for the kubelet's initial bootstrap credentials, the following two authenticators are recommended for ease of provisioning.

1. [Bootstrap Tokens](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/#bootstrap-tokens)
2. [Token authentication file](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/#token-authentication-file)

Using bootstrap tokens is a simpler and more easily managed method to authenticate kubelets, and does not require any additional flags when starting kube-apiserver.

Whichever method you choose, the requirement is that the kubelet be able to authenticate as a user with the rights to:

1. create and retrieve CSRs
2. be automatically approved to request node client certificates, if automatic approval is enabled.

## Bootstrap tokens
Bootstrap tokens are described in detail [here](https://kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/). These are tokens that are stored as secrets in the Kubernetes cluster, and then issued to the individual kubelet. You can use a single token for an entire cluster, or issue one per worker node.

The process is two-fold:

1. Create a Kubernetes secret with the token ID, secret and scope(s).
2. Issue the token to the kubelet

From the kubelet's perspective, one token is like another and has no special meaning. From the kube-apiserver's perspective, however, the bootstrap token is special. Due to its `type`, `namespace` and `name`, kube-apiserver recognizes it as a special token, and grants anyone authenticating with that token special bootstrap rights, notably treating them as a member of the `system:bootstrappers` group. This fulfills a basic requirement for TLS bootstrapping.

The details for creating the secret are available [here](https://kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/).

If you want to use bootstrap tokens, you must enable it on kube-apiserver with the flag:

```console
--enable-bootstrap-token-auth=true
```



（Certificate Rotation）是指定期更新和替换数字证书的过程。在Kubernetes的上下文中，特别是对于kubelet，这意味着kubelet用来进行通信和身份验证的证书会被自动或半自动地更新。

让我们分解一下这个概念：

- **证书（Certificates）**：数字证书是一种电子凭证，用于验证某个实体（例如服务器、客户端或个人）的身份，并通常包含一个公钥。它们在加密通信（如TLS/SSL）中发挥关键作用，确保数据的机密性、完整性和真实性。
    
- **轮转（Rotation）**：就像我们定期更换家里的空气过滤器或汽车的机油一样，证书轮转就是定期更换或更新证书。这不是一个一次性的事件，而是一个持续的维护过程。
    

**为什么需要证书轮转？**

证书轮转对于维护系统安全至关重要，主要有以下几个原因：

1. **减少密钥泄露的风险**：如果一个证书的私钥被泄露，攻击者就可以冒充证书的合法所有者。定期轮转证书可以限制私钥泄露的影响范围和时间窗口。即使私钥被泄露，一旦证书过期并被替换，该泄露的私钥就失去了作用。
    
2. **强制性安全最佳实践**：许多安全标准和合规性要求规定了证书的有效期和轮转策略。
    
3. **防止证书过期**：所有证书都有一个有效期。如果不定期轮转，证书就会过期，导致服务中断，因为依赖该证书的通信将无法建立。自动轮转可以避免手动跟踪和更新证书的繁琐工作，从而防止意外过期。
    
4. **应对密码学发展**：随着密码学研究的进展和计算能力的提升，某些加密算法或密钥长度可能会变得不安全。定期轮转证书可以允许系统在必要时切换到更强的加密算法和更大的密钥长度。
    

**Kubernetes中的证书轮转（特别是kubelet）**

如您引用的文字所述，Kubernetes v1.8及更高版本中的kubelet实现了证书轮转功能，用于其客户端和/或服务证书。

- **客户端证书（Client Certificates）**：kubelet使用客户端证书向Kubernetes API服务器进行身份验证。通过轮转这些证书，可以确保kubelet与API服务器之间的通信始终使用最新的、安全的凭证。
    
- **服务证书（Serving Certificates）**：kubelet还提供一个HTTPS端点，用于API服务器或其他组件与其进行通信（例如，获取节点状态、执行命令等）。服务证书用于此HTTPS连接的加密和身份验证。如您所指，服务证书的轮转是一个beta功能（`RotateKubeletServerCertificate` 特性标志，默认启用），它增强了kubelet自身安全通信的韧性。
    

总而言之，证书轮转是一项重要的安全实践，它通过定期更新和替换数字证书来降低安全风险、防止服务中断并适应不断变化的安全需求。在Kubernetes中，它使得kubelet能够自动管理其通信凭证，从而提高了集群的整体安全性。


你提到 Kubelet 在引导（bootstrap）阶段没有完全的权限，这是非常正确的。这是一个关键的安全机制，被称为 **TLS 引导（TLS Bootstrapping）**，它确保了新加入集群的 Kubelet 在获得完全权限之前，身份是受控的。

以下是 Kubelet 如何逐步获得足够权限的完整过程：

---

### Kubelet 引导阶段的权限限制

在 Kubelet 刚启动时，它需要与 **Kubernetes API Server** 进行通信，但它还没有有效的客户端证书来证明自己的身份。因此，它不能随意执行操作。它拥有的仅仅是**引导凭证（bootstrap credentials）**，通常是一个短期的**引导令牌（bootstrap token）**。

这个引导令牌只赋予 Kubelet **非常有限的权限**，通常包括：

1. **创建证书签名请求（CSR）的权限：** Kubelet 能够向 API Server 发送一个请求，要求为自己签发一个客户端证书。它会将自己的公钥包含在 CSR 中。
    
2. **获取自身 CSR 状态的权限：** Kubelet 需要能够查询它之前提交的 CSR 的状态，以等待其被批准。
    

Kubelet 在这个阶段绝对**不能**执行以下操作：

- **创建、更新、删除 Pods、Services、Deployments 等集群资源。**
    
- **读取敏感信息**（如 Secrets）。
    
- **执行任何可能影响集群安全的操作。**
    

---

### Kubelet 获得足够权限的完整过程

Kubelet 获得其完整权限的过程是一个自动化的证书签发和身份验证流程：

#### 1. 引导 Kubeconfig 文件

在 Kubelet 启动时，它会通过 `--bootstrap-kubeconfig` 参数指定一个特殊的 Kubeconfig 文件。这个文件包含：

- **API Server 的地址：** Kubelet 知道要连接哪个 API Server。
    
- **引导令牌：** 这是一个预先配置好的、具有有限权限的令牌。
    

#### 2. Kubelet 发送证书签名请求 (CSR)

- Kubelet 使用这个引导 Kubeconfig 文件中的令牌向 API Server 进行身份验证。
    
- 它利用其有限的权限，生成一个客户端密钥对，并使用公钥创建一个 **证书签名请求（Certificate Signing Request, CSR）**，然后将这个 CSR 发送给 API Server。这个 CSR 请求通常会带有特殊的 `usages`，表明它是一个节点客户端证书。
    

#### 3. API Server 接收 CSR 并等待批准

- API Server 接收到 Kubelet 的 CSR。
    
- API Server 会将这个 CSR 存储起来，并根据 RBAC 策略验证提交 CSR 的引导令牌是否有权创建 CSR。由于引导令牌通常被授予 `system:node-bootstrapper` 角色，因此这个步骤会通过。
    

#### 4. CSR 的批准

这是 Kubelet 权限提升的关键一步：

- **自动批准（推荐且常见）：** 在生产环境中，通常会配置 `kube-controller-manager`（或使用 `kubeadm` 等工具配置的自动批准控制器）来自动批准来自特定组（例如 `system:bootstrappers`）的 Kubelet CSR。这意味着只要 CSR 符合预设的条件（例如，来自一个引导令牌），它就会被自动批准。
    
- **手动批准（仅用于调试或特殊情况）：** 集群管理员也可以通过 `kubectl get csr` 和 `kubectl certificate approve <csr-name>` 命令手动审查和批准 CSR。
    

一旦 CSR 被批准，API Server 就会使用集群的证书颁发机构（CA）来**签发一个真正的客户端证书**，并将其与 Kubelet 的公钥关联起来。

#### 5. Kubelet 获取签发的证书

- Kubelet 会定期查询它之前提交的 CSR 的状态。
    
- 一旦发现 CSR 状态变为“已批准”并包含了签发的证书，Kubelet 就会下载这个证书。
    

#### 6. Kubelet 更新 Kubeconfig 并获得完整权限

- Kubelet 会将新获得的客户端证书和对应的私钥写入到其 `--kubeconfig` 参数指定的**正式 Kubeconfig 文件**中。
    
- 从这一刻起，Kubelet 不再使用引导令牌进行身份验证。它将使用新获得的**客户端证书**来与 API Server 进行通信。
    
- 这个客户端证书通常会赋予 Kubelet 完整的**节点权限**，因为它现在作为集群中的一个“已注册节点”被信任。这些权限包括：
    
    - **读写其节点上的 Pods 信息。**
        
    - **报告节点状态和容量。**
        
    - **挂载/卸载存储卷。**
        
    - **创建事件等。**
        
    - **管理容器运行时。**
        

#### 7. 证书轮转（可选但推荐）

为了进一步增强安全性，Kubelet 还可以配置为**自动轮转其客户端证书**（以及实验性的服务证书）。这意味着在证书过期之前，Kubelet 会重复步骤 2-6，请求新的证书，确保其凭证始终有效且不会因过期而导致服务中断。

---

**总结：**

Kubelet 在启动时，仅使用一个**有限权限的引导令牌**连接到 API Server。它的主要目标是利用这个令牌**请求并获取一个由集群 CA 签发的、具有完整节点权限的客户端证书**。一旦这个证书被签发并配置到 Kubelet 的正式 Kubeconfig 文件中，Kubelet 就获得了执行其所有节点管理任务所需的**完整权限**。这个引导和证书签发过程是 Kubernetes 节点加入集群时安全的关键保障。


Kubelet 服务器证书（Serving Certificate）的签发时机主要发生在 Kubelet **客户端证书引导完成之后，并且是在证书即将过期时进行轮转**。这个过程与 Kubelet 客户端证书的获取紧密相关。

以下是 Kubelet 服务器证书签发和轮转的时机：

---

### 1. 初次签发：客户端证书引导成功之后

在 Kubelet 引导过程（TLS Bootstrapping）中，首先获得的是**客户端证书**。这个客户端证书用于 Kubelet 向 API Server 进行认证，证明自己的身份。

一旦 Kubelet 成功获取并配置了其客户端证书，它就获得了足够的权限来与 API Server 进行安全通信。此时，如果 **`RotateKubeletServerCertificate` 特性门（feature gate）被启用（Kubernetes v1.8+ 默认启用）**，Kubelet 就会：

- **发起服务器证书请求：** Kubelet 会自动生成一个新的服务器密钥对，并使用其客户端证书向 API Server 发送一个证书签名请求（CSR），请求签发一个用于其 HTTPS 服务器（通常是端口 10250）的证书。这个 CSR 会明确指示其用途为“服务器认证”（Server Authentication）。
    
- **API Server 批准并签发：** 类似于客户端证书的批准过程，API Server（通常由 `kube-controller-manager` 中的证书控制器自动批准）会验证这个 CSR，并使用集群的 CA 签发一个服务器证书。
    
- **Kubelet 获取并使用：** Kubelet 下载这个签发的服务器证书，并将其配置为 Kubelet HTTPS 服务器所使用的证书。从此以后，API Server 和其他组件在与 Kubelet 的 10250 端口进行安全通信时，就会使用这个证书进行验证。

**安全通信的建立：**

- 从此以后，当 API Server 或其他 Kubernetes 组件（如 `kube-proxy`、`kubectl` 等）需要与 Kubelet 的 10250 端口进行安全通信时，它们会发起 HTTPS 连接。
    
- 在 TLS 握手过程中，Kubelet 会将其配置的服务器证书发送给发起连接的客户端。
    
- 客户端会使用集群的根 CA 证书来验证 Kubelet 提供的服务器证书。如果证书是由受信任的根 CA 签发的，并且证书中的主机名与 Kubelet 的实际主机名匹配，客户端就会信任 Kubelet 的身份，并建立安全的通信通道。

---

### 2. 证书轮转：在当前证书过期前

Kubelet 服务器证书并非一次性签发就永久有效。为了维护安全性并防止证书过期导致服务中断，Kubelet 实现了证书轮转机制。

- **定期检查：** Kubelet 会定期检查其当前服务器证书的有效期。
    
- **接近过期时触发：** 当 Kubelet 检测到其服务器证书即将过期（通常是在证书生命周期的大约 70%-80% 处，或者距离过期只剩几天时），它会主动触发一个新的证书签发流程。
    
- **重复请求过程：** Kubelet 会重复初次签发时的步骤：
    
    1. 生成新的服务器密钥对。
        
    2. 向 API Server 发送新的 CSR。
        
    3. API Server 批准并签发新的服务器证书。
        
    4. Kubelet 获取新证书。
        
- **替换旧证书：** 一旦新证书可用，Kubelet 会在运行时无缝地替换旧证书，而无需重启服务。这通常通过在内存中加载新证书并更新 HTTPS 服务器的配置来实现，以确保连接不会中断。
    

---

### 关键点总结：

- **依赖客户端证书：** Kubelet 服务器证书的获取，依赖于 Kubelet 已经成功获得了其客户端证书，因为需要使用客户端证书向 API Server 进行认证并发送 CSR。
    
- **`RotateKubeletServerCertificate` 特性门：** 这是控制服务器证书自动轮转的关键开关，在 Kubernetes v1.8 及更高版本中默认启用。
    
- **自动管理：** 整个服务器证书的签发和轮转过程是高度自动化的，旨在减少集群管理员的负担并提高集群的安全性。
    
- **防止中断：** 自动轮转机制确保了 Kubelet 的 HTTPS 服务器能够始终使用有效的证书，从而避免因证书过期而导致的通信中断。
    

因此，你可以认为 Kubelet 服务器证书的签发时机是：**在 Kubelet 完成客户端引导并获得认证能力后首次进行，此后在现有服务器证书接近过期时进行自动轮转续签。**