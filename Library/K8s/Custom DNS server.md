## 自定义 DNS 服务器 (Custom DNS Server)
在企业网络环境中，**自定义 DNS 服务器 (Custom DNS Server)** 是一个非常重要的概念。它指的是组织内部部署和管理的 DNS (Domain Name System) 服务器，而不是使用公共的 DNS 服务（如 Google 的 8.8.8.8 或 Cloudflare 的 1.1.1.1）。

---

### **为什么企业需要自定义 DNS 服务器？**

企业采用自定义 DNS 服务器的主要原因包括：

1. **内部域名解析**: 企业内部有大量的服务、应用程序和资源可能不通过公共 DNS 解析。例如，`intranet.yourcompany.com`、`jira.internal` 或内部数据库的地址。自定义 DNS 服务器能够解析这些只有在企业内部才能访问的私有域名。
    
2. **安全性与控制**: 企业可以完全控制 DNS 查询的路由和结果。这有助于防止 DNS 劫持、恶意域名解析，并可以强制执行安全策略，例如阻止访问已知的恶意网站。
    
3. **性能优化**: 对于频繁访问的内部资源，通过内部 DNS 服务器解析可以减少延迟，提高访问速度。
    
4. **分流和流量管理**: 可以根据策略将流量导向特定的内部服务实例，实现负载均衡或地理位置路由。
    
5. **隐私保护**: 内部 DNS 查询不会泄露给外部公共 DNS 提供商，增强了数据隐私。
    
6. **合规性**: 某些行业或法规可能要求企业对其网络基础设施拥有完全的控制权，包括 DNS 解析。
    

---

### **自定义 DNS 服务器在 Kubernetes 中的作用**

在 Kubernetes 集群中，自定义 DNS 服务器扮演着至关重要的角色，尤其是在企业网络环境下：

1. **CoreDNS 的上游解析**: Kubernetes 默认使用 **CoreDNS** 作为集群内部的 DNS 服务。CoreDNS 负责解析集群内部的服务（`my-service.my-namespace.svc.cluster.local`）和 Pod 的 IP 地址。然而，对于那些不是集群内部的域名（例如 `www.google.com` 或 `jira.yourcompany.com`），CoreDNS 需要将这些查询转发给**上游 DNS 服务器**进行解析。在企业环境中，这个上游 DNS 服务器就是你的**自定义 DNS 服务器**。
    
2. **主机节点解析**: 集群中的每个工作节点（包括 Master 节点和 Worker 节点）自身的操作系统也需要能够正确解析域名。例如，kubelet 可能需要解析镜像仓库的域名来拉取镜像，或者节点需要解析外部服务地址。这些节点通常会配置为使用企业的自定义 DNS 服务器。
    
3. **Pod 访问内部资源**: 如果你的 Pods 需要访问企业内部的资源（如内部数据库、内部 API 服务、内部文件共享等），而这些资源只有通过企业内部 DNS 才能解析，那么 CoreDNS 必须能够正确地将这些查询转发到你的自定义 DNS 服务器。
    

---

### **配置示例**

假设你的企业自定义 DNS 服务器的 IP 地址是 `10.0.0.10` 和 `10.0.0.11`，并且你的内部域名是 `yourcompany.com`。

#### **1. 配置 CoreDNS 使用自定义 DNS 服务器**

你需要修改 Kubernetes 中 `kube-system` 命名空间下的 `coredns` ConfigMap。

首先，编辑 ConfigMap：

```
kubectl edit configmap coredns -n kube-system
```

找到 `Corefile` 配置块，并添加或修改 `forward` 插件，指向你的自定义 DNS 服务器。确保 `forward` 规则位于 `kubernetes` 规则的下方，这样 CoreDNS 会优先解析集群内部域名，然后再转发外部域名。

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
        }
        # 将这里的 10.0.0.10 和 10.0.0.11 替换为你公司内部的 DNS 服务器 IP 地址
        # `prefer_udp` 是推荐的设置，优先使用 UDP 协议进行查询
        forward . 10.0.0.10 10.0.0.11 {
           prefer_udp
        }
        cache 30
        loop
        reload
        loadbalance
    }
```

保存并退出编辑器后，CoreDNS Pods 会自动更新其配置或重启以应用更改。

#### **2. 配置主机节点使用自定义 DNS 服务器**

集群中的每个节点都需要配置 `/etc/resolv.conf` 文件，以便它能够正确解析域名。

```
# /etc/resolv.conf 示例 (在每个 Kubernetes 节点上)
nameserver 10.0.0.10      # 你的主自定义 DNS 服务器 IP
nameserver 10.0.0.11      # 你的备用自定义 DNS 服务器 IP
search yourcompany.com    # 添加你的企业内部搜索域
domain yourcompany.com    # 可选，如果适用
```

**注意**: 在某些 Linux 发行版上，`systemd-resolved` 或 `NetworkManager` 可能会覆盖 `/etc/resolv.conf`。在这种情况下，你需要通过这些服务来配置 DNS 设置，而不是直接编辑 `/etc/resolv.conf`。

- **对于 `systemd-resolved`**: 编辑 `/etc/systemd/resolved.conf` 文件，取消注释并设置 `DNS=` 和 `Domains=`。

    ```
    # /etc/systemd/resolved.conf
    [Resolve]
    DNS=10.0.0.10 10.0.0.11
    Domains=yourcompany.com
    ```
    
    然后重启 `systemd-resolved` 服务：`sudo systemctl restart systemd-resolved`。
    
- **对于 `NetworkManager`**: 通常通过网络配置文件（如 `/etc/sysconfig/network-scripts/ifcfg-eth0` 或 `/etc/NetworkManager/system-connections/your_connection`）来设置 DNS。
    
---

你提供的 `Corefile` 是 **CoreDNS** 的核心配置文件，它定义了 CoreDNS 如何处理 DNS 查询请求。每个块（以 `.:53 { ... }` 开头）代表一个 **CoreDNS 插件链**，用于处理特定端口和域名的查询。

下面我将详细解释你提供的 `Corefile` 片段中每个字段的含义：

---

### `.:53 { ... }`

这是一个 **Server Block**，表示 CoreDNS 将监听在所有网络接口的 **53 端口**上，并响应所有域名的 DNS 查询。`:` 表示监听所有 IP 地址，`.` 表示匹配所有域名。

---

### **插件解释**

`Corefile` 内部的每一行通常代表一个 **CoreDNS 插件**，它们按照从上到下的顺序处理 DNS 查询。

- **`errors`**
    
    - **含义**: 这个插件会记录所有在处理 DNS 查询过程中发生的错误。
        
    - **作用**: 当 CoreDNS 无法解析域名、遇到内部错误或与其他 DNS 服务器通信失败时，它会将错误信息记录到日志中。这对于调试和监控 CoreDNS 的运行状态非常有用。
        
- **`health { lameduck 5s }`**
    
    - **含义**: 这是一个健康检查插件。它会响应 HTTP 请求，报告 CoreDNS 服务的健康状态。
        
        - `lameduck 5s`: 这个参数表示在 CoreDNS Pod 收到终止信号后，它将继续处理现有请求 **5 秒**，然后再关闭。这被称为“优雅关闭”或“瘸鸭模式”，可以确保在 Pod 被销毁之前，它有时间完成正在处理的请求，并从服务发现中注销，从而避免请求丢失。
            
    - **作用**: 通常被 Kubernetes 的 Liveness 和 Readiness Probes (存活探针和就绪探针) 使用，以确定 CoreDNS Pod 是否健康并可以接收流量。
        
- **`ready`**
    
    - **含义**: 这是一个就绪探针插件。它在 CoreDNS 准备好接收流量时响应 `/ready` HTTP 请求。
        
    - **作用**: Kubernetes 的 Readiness Probes 会查询这个端点。只有当 CoreDNS 认为自己完全准备好并已加载所有配置时，它才会通过这个探针。这有助于防止流量在 CoreDNS 尚未完全启动时就被路由过来。
        
- **`kubernetes cluster.local in-addr.arpa ip6.arpa { ... }`**
    
    - **含义**: 这是 CoreDNS 的核心插件，用于处理 Kubernetes 集群内部的 DNS 解析。
        
        - `cluster.local`: 这是 Kubernetes 集群的服务域名后缀（Service Domain）。所有 Kubernetes 内部服务的 DNS 查询都会以 `*.svc.cluster.local` 或 `*.pod.cluster.local` 结尾。
            
        - `in-addr.arpa`: 用于 IPv4 地址的反向 DNS 查询 (PTR 记录)。例如，将 IP 地址转换为域名。
            
        - `ip6.arpa`: 用于 IPv6 地址的反向 DNS 查询。
            
        - `pods insecure`: 这个选项允许 CoreDNS 为 Pods 进行 DNS 解析，即使 Pod 没有关联的服务。`insecure` 关键字表示即使 Pod 不在你的集群域中，也会解析其 IP。
            
        - `fallthrough in-addr.arpa ip6.arpa`: 这表示如果 CoreDNS 无法解析 `in-addr.arpa` 或 `ip6.arpa` 域名的查询，它会将这些查询传递给下一个插件处理（即 `forward` 插件）。
            
    - **作用**: 使 Pod 和 Services 能够相互发现和通信。
        
- **`forward . 10.0.0.10 10.0.0.11 { prefer_udp }`**
    
    - **含义**: 这是一个转发插件，用于将 CoreDNS 自身无法解析的查询转发给其他上游 DNS 服务器。
        
        - `.`: 表示匹配所有剩余的域名查询（即不属于 `cluster.local` 等内部域名的查询）。
            
        - `10.0.0.10 10.0.0.11`: 这是你配置的**自定义 DNS 服务器**的 IP 地址列表。CoreDNS 会将查询转发给这些服务器。
            
        - `prefer_udp`: 这个选项表示 CoreDNS 在转发查询时优先使用 UDP 协议。DNS 查询通常使用 UDP，只有在响应过大或需要可靠传输时才回退到 TCP。
            
    - **作用**: 允许 Kubernetes Pods 和 Services 访问集群外部的域名，例如公共互联网上的网站（如 `www.google.com`）或你企业内部的私有服务（如 `jira.yourcompany.com`）。
        
- **`cache 30`**
    
    - **含义**: 这是一个缓存插件。它会缓存 CoreDNS 响应的 DNS 查询结果。
        
        - `30`: 表示默认的 TTL (Time-To-Live) 是 30 秒，即如果原始 DNS 记录没有指定 TTL，缓存将保留 30 秒。
            
    - **作用**: 提高 DNS 查询的性能，减少对上游 DNS 服务器的请求量。对于重复查询，CoreDNS 可以直接从缓存中返回结果，加快响应速度。
        
- **`loop`**
    
    - **含义**: 这是一个循环检测插件。它用于检测并防止 DNS 查询在 CoreDNS 自身或与上游服务器之间形成无限循环。
        
    - **作用**: 如果 CoreDNS 发现查询可能陷入循环，它会中断查询并记录错误，防止资源耗尽。
        
- **`reload`**
    
    - **含义**: 这是一个热重载插件。它允许 CoreDNS 在其 `ConfigMap` 配置发生变化时自动重新加载 `Corefile`。
        
    - **作用**: 你无需手动重启 CoreDNS Pod 就可以应用配置更改。当 `coredns` ConfigMap 更新后，CoreDNS Pod 会检测到变化并自动加载新配置。
        
- **`loadbalance`**
    
    - **含义**: 这是一个负载均衡插件。它在 CoreDNS 的多个实例之间（如果部署了多个 CoreDNS Pod）以及在转发查询到多个上游 DNS 服务器时，提供简单的 DNS 负载均衡。
        
    - **作用**: 确保 DNS 查询请求均匀地分布在 CoreDNS 实例或上游服务器之间，提高可用性和性能。
        

---

总的来说，这个 `Corefile` 配置定义了一个健壮的 CoreDNS 设置，它既能处理 Kubernetes 集群内部的 DNS 解析，又能将外部解析请求转发到你指定的企业内部 DNS 服务器，同时还包含了错误日志、健康检查、缓存、循环防护和热重载等功能，以确保 DNS 服务的稳定性和效率。