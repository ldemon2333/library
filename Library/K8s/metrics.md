在 Kubernetes 中查看 Metrics（度量指标）信息，通常分为几种类型和不同的查看方式。理解不同类型指标的来源和用途是关键。

---

### 1. 资源使用指标 (Resource Usage Metrics)

这类指标主要包括 Pod 和 Node 的 CPU、内存使用情况。它们是 Kubernetes 内置的自动扩缩容（HPA、VPA）和 `kubectl top` 命令的基础。

#### a. 使用 `kubectl top` (最常用和便捷)

这是查看 Pod 和 Node 实时（或接近实时）资源使用情况的最快捷方式。

- **查看节点资源使用：**
    
    ```
    kubectl top nodes
    ```
    
    输出示例：
    
    ```
    NAME         CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
    k8s-node-1   120m         6%     1234Mi          30%
    k8s-node-2   80m          4%     987Mi           25%
    ```
    
- **查看 Pod 资源使用：**
    

    ```
    kubectl top pods
    ```
    
    输出示例：
    
    ```
    NAME                      CPU(cores)   MEMORY(bytes)
    my-app-pod-abcde          5m           20Mi
    nginx-deployment-fghij    2m           10Mi
    ```
    
- **查看特定命名空间下的 Pod：**
    

    ```
    kubectl top pods -n my-namespace
    ```
    
- **查看所有命名空间下的 Pod：**
    

    ```
    kubectl top pods --all-namespaces
    ```
    

**注意：** `kubectl top` 命令依赖于集群中部署的 **Metrics Server**。如果 `kubectl top` 命令报错，很可能是你的集群中没有安装或 Metrics Server 没有正常运行。

```
* **检查 Metrics Server 是否运行：**
    ```bash
    kubectl get deployment metrics-server -n kube-system
    ```
* **安装 Metrics Server (如果未安装)：**
    ```bash
    kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
    ```
    （请根据你的 Kubernetes 版本选择合适的 `components.yaml` 版本）
```

#### b. 直接查询 Metrics API (高级)

Metrics Server 会将收集到的资源指标通过 **Metrics API (`metrics.k8s.io/v1beta1`)** 暴露给 API Server。你可以直接查询这个 API。

- **查询所有节点指标：**
    
    Bash
    
    ```
    kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes" | jq .
    ```
    
- **查询特定节点指标：**
    
    Bash
    
    ```
    kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes/<node-name>" | jq .
    ```
    
- **查询所有 Pod 指标：**
    
    Bash
    
    ```
    kubectl get --raw "/apis/metrics.k8s.io/v1beta1/pods" | jq .
    ```
    
- **查询特定 Pod 指标：**
    
    Bash
    
    ```
    kubectl get --raw "/apis/metrics.k8s.io/v1beta1/namespaces/<namespace>/pods/<pod-name>" | jq .
    ```
    
    `jq` 是一个命令行 JSON 处理器，用于美化输出，方便阅读。
    

---

### 2. 控制平面和组件指标 (Control Plane & Component Metrics)

Kubernetes 的核心组件（kube-apiserver, kube-controller-manager, kube-scheduler, kubelet, kube-proxy 等）自身会暴露大量的内部操作指标，通常采用 **Prometheus 格式**。这些指标提供了组件的健康状况、请求延迟、错误率等详细信息。

#### a. 直接访问组件的 `/metrics` 端点

许多 Kubernetes 组件会在其监听端口上暴露一个 `/metrics` HTTP 端点。你可以通过 `kubectl port-forward` 或直接访问节点 IP 来查看。

- 查看 Kubelet 指标 (节点级别)：
    
    Kubelet 在每个节点上运行，它暴露了多种指标端点：
    
    - `/metrics`: Kubelet 自身的 Prometheus 格式指标。
        
    - `/metrics/cadvisor`: cAdvisor 收集的容器和节点底层资源指标。
        
    - `/metrics/resource`: Kubelet 提供的资源指标，供 Metrics Server 抓取。
        
    - `/metrics/probes`: 有关就绪性/存活探针的指标。
        
    
    **通过 `kubectl port-forward` 查看 Kubelet 指标：**
    
    Bash
    
    ```
    # 找到 Kubelet Pod 的名称，通常是 daemonset 管理的
    kubectl get pods -n kube-system | grep kube-proxy # 找到某个节点上的kube-proxy pod名
    # 或者直接找到kubelet的进程监听的端口（通常是10255或10250）
    
    # 假设你找到了一个 kube-proxy pod (它与kubelet在同一节点)
    # 或者直接用`kubectl get node <node-name> -o yaml`看kubelet的地址
    
    # 假设Kubelet监听在节点IP:10255 (secure port) 或 10250 (read-only port)
    # 你可能需要配置RBAC才能访问这些端口
    # 更简便的方法是使用 `kubectl proxy` 访问 API Server 再转发
    ```
    
    更简单且更安全的方法（如果组件暴露给 API Server）：
    
    对于某些控制平面组件（如 kube-apiserver, kube-controller-manager），如果你能访问其 Pod，可以使用 kubectl port-forward 或通过 API Server 代理访问。
    
    例如，通过 `kubectl proxy` (会启动一个本地代理到 Kubernetes API Server)：
    
    Bash
    
    ```
    kubectl proxy --port=8001
    ```
    
    然后通过浏览器访问：
    
    - `http://localhost:8001/api/v1/namespaces/kube-system/pods/<kube-apiserver-pod-name>/proxy/metrics`
        
    - `http://localhost:8001/api/v1/namespaces/kube-system/pods/<kube-controller-manager-pod-name>/proxy/metrics`
        

#### b. 使用专业的监控系统 (推荐用于生产环境)

对于长期存储、高级查询、可视化和告警，你需要一个专业的监控解决方案。

- **Prometheus + Grafana (业界标准)：**
    
    - **Prometheus:** 一个开源的监控系统，专门用于收集和存储时序数据。它可以配置为抓取（scrape）Kubernetes 各组件的 `/metrics` 端点，以及应用程序自定义的 metrics。
        
    - **Grafana:** 一个开源的数据可视化和仪表板工具。它可以连接到 Prometheus 作为数据源，并创建丰富的图表和仪表板来展示 Kubernetes 集群的各项指标。
        
    - 通常通过 **kube-prometheus-stack** (Helm Chart) 部署，它包含了 Prometheus Operator, Prometheus, Grafana, Alertmanager, kube-state-metrics 等，提供了一站式的 Kubernetes 监控方案。
        
- **其他商业或云原生的监控解决方案：**
    
    - Datadog, New Relic, Dynatrace 等 APM 工具。
        
    - 云服务提供商的监控服务，如 AWS CloudWatch, Google Cloud Monitoring, Azure Monitor 等，它们通常与 Kubernetes 服务深度集成。
        

---

### 3. Kubernetes 状态指标 (Kubernetes State Metrics)

这类指标关注 Kubernetes 对象的生命周期状态，例如：

- Pod 的运行状态、重启次数、未就绪数量。
    
- Deployment、DaemonSet、StatefulSet 的期望副本数、当前可用副本数。
    
- Node 的状态、文件系统使用情况。
    

这些指标由 **kube-state-metrics** 服务收集并以 Prometheus 格式暴露。它与 Metrics Server 不同，Metrics Server 关注资源使用，而 `kube-state-metrics` 关注 Kubernetes 对象本身的状态。

- 查看方法：
    
    kube-state-metrics 通常也以 Pod 形式运行在 kube-system 命名空间下。你可以像查看其他组件指标一样，通过 kubectl port-forward 或集成到 Prometheus 中来抓取其 /metrics 端点。
    

---

### 总结

查看 Kubernetes metrics 的方式取决于你想要获取的指标类型和你的需求：

- **快速概览 Pod/Node CPU/内存使用：** 使用 **`kubectl top`** (需要 Metrics Server)。
    
- **深入分析组件内部性能或自定义应用指标：** 需要部署 **Prometheus/Grafana** 等监控系统，抓取各组件的 `/metrics` 端点。
    
- **获取 Kubernetes 对象状态指标：** 需要部署 **kube-state-metrics**，并通过 Prometheus 等工具抓取其 `/metrics` 端点。
    

对于生产环境，强烈建议部署完整的监控解决方案，如 **Prometheus + Grafana**，以便进行全面的集群健康和性能监控。