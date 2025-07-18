## Kubelet 的 Standalone 模式

Kubelet 主要作为 Kubernetes 集群中的一个节点代理运行，但它也提供了一个**Standalone 模式**（或称作**单机模式**）。在这种模式下，Kubelet 不会尝试连接到 Kubernetes API Server 来获取 Pod 定义或注册节点。相反，它会直接从本地文件或 HTTP 端点读取 Pod 定义，并在本地机器上运行这些 Pod。

这种模式的主要目的是**在没有完整 Kubernetes 控制平面（例如 Master 节点）的情况下，单独运行容器化工作负载**。它通常用于以下场景：

- **边缘计算 (Edge Computing)**：在资源受限或网络连接不稳定的边缘设备上，部署完整的 Kubernetes 集群可能不切实际。Kubelet Standalone 模式允许在这些设备上独立运行关键应用。
    
- **本地开发和测试**：开发者可以在自己的机器上快速启动和测试容器化应用，而无需搭建一个完整的 Kubernetes 集群（例如 Minikube 或 Kind）。
    
- **简单的单机部署**：对于只需要在单个服务器上运行少量容器，且不需要集群管理、调度、服务发现等高级功能的场景，Standalone Kubelet 是一个轻量级的选择。
    
- **IoT 设备**：在物联网设备上运行容器，Kubelet 可以作为轻量级的容器管理器。
    

---

### 工作原理

在 Standalone 模式下，Kubelet 不会通过 `kubeconfig` 文件与 API Server 通信。取而代之的是，它会通过以下两种主要的 Pod 定义来源来获取需要运行的 Pod 信息：

1. **静态 Pods (Static Pods)**：
    
    - Kubelet 会监视一个本地目录（通过 `--pod-manifest-path` 参数指定，例如 `/etc/kubernetes/manifests`）。
        
    - 当该目录下出现 `.yaml` 或 `.json` 格式的 Pod 定义文件时，Kubelet 会解析这些文件，并负责在本地机器上创建、运行和管理这些 Pod。
        
    - 如果这些文件被修改或删除，Kubelet 也会相应地更新或终止 Pod。
        
    - 这种方式创建的 Pod 不会被 API Server 管理（因为 Kubelet 不连接 API Server），因此它们不会出现在 `kubectl get pods` 的输出中（除非 Kubelet 也在连接 API Server 并将它们注册为镜像 Pod）。
        
2. **HTTP 端点**：
    
    - Kubelet 可以配置从一个特定的 HTTP URL（通过 `--manifest-url` 参数指定）定期拉取 Pod 定义。
        
    - 这种方式使得 Pod 定义可以集中管理，并通过 Web 服务器分发给多个 Standalone Kubelet 实例。
        

---

### 限制

虽然 Standalone 模式提供了一定的灵活性，但它也有明显的限制：

- **无集群功能**：Standalone Kubelet 不参与 Kubernetes 集群的任何高级功能，例如：
    
    - **调度 (Scheduling)**：没有调度器，Pod 只能运行在 Kubelet 所在的节点上。
        
    - **服务发现 (Service Discovery)**：没有 Kube-proxy 或 CoreDNS，内部服务发现不可用。
        
    - **负载均衡 (Load Balancing)**：没有内建的负载均衡机制。
        
    - **自动扩缩容 (Autoscaling)**：无法进行 Pod 或节点的自动扩缩容。
        
    - **持久存储 (Persistent Storage)**：虽然 Kubelet 可以挂载卷，但没有 CSI 驱动和动态存储供应。
        
    - **Secrets/ConfigMaps 管理**：虽然 Pod 可以引用它们，但没有 API Server 集中管理。
        
- **单点故障**：如果运行 Standalone Kubelet 的节点出现故障，其上的工作负载也会停止，没有高可用性保障。
    
- **管理复杂性**：当节点数量增加时，手动管理每个 Kubelet 实例及其 Pod 定义会变得非常复杂。
    

---

### 如何启动 Standalone Kubelet

要以 Standalone 模式运行 Kubelet，你通常会省略 `kubeconfig` 相关的参数，并指定 Pod 定义的来源：

Bash

```
# 假设你的 Pod 定义文件放在 /etc/kubernetes/manifests/my-app.yaml
sudo kubelet \
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \
  --pod-manifest-path=/etc/kubernetes/manifests \
  --allow-privileged=true \
  --read-only-port=0 \
  --cgroup-driver=systemd \
  --network-plugin=cni \
  --hostname-override=<your-node-name> \
  --feature-gates=RotateKubeletClientCertificate=false,RotateKubeletServerCertificate=false
```

**解释：**

- `--container-runtime-endpoint`: 指定容器运行时（如 containerd 或 Docker）的套接字路径。
    
- `--pod-manifest-path`: 指定静态 Pod 定义文件所在的目录。Kubelet 会监视这个目录。
    
- `--read-only-port=0`: 禁用 Kubelet 的只读 API 端口，增加安全性。
    
- `--cgroup-driver`: 指定 cgroup 驱动，通常与操作系统的 cgroup 驱动保持一致 (systemd 或 cgroupfs)。
    
- `--network-plugin=cni`: Kubelet 仍然需要网络插件来为 Pod 配置网络，即使在 Standalone 模式下。
    
- `--hostname-override`: 覆盖 Kubelet 使用的 hostname，对于节点命名很重要。
    
- `--feature-gates=RotateKubeletClientCertificate=false,RotateKubeletServerCertificate=false`: 由于不连接 API Server，禁用证书轮换功能。
    

---

总的来说，Kubelet 的 Standalone 模式提供了一个在非集群环境中运行容器化工作负载的轻量级解决方案。它适用于特定的用例，但牺牲了 Kubernetes 集群提供的强大管理和自动化能力。