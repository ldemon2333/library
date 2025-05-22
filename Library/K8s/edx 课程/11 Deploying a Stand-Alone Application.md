In this chapter, we will learn how to deploy an application using the **Dashboard (Kubernetes WebUI)** and the **Command Line Interface (CLI)**. We will also expose the application with a NodePort type Service, and access it from outside the Minikube cluster.

By the end of this chapter, you should be able to:

- Deploy an application from the dashboard.
- Deploy an application from a YAML file using kubectl.
- Expose a service using NodePort.
- Access the application from outside the Minikube cluster.

**$ kubectl describe service web-service**
![[Pasted image 20250515150822.png]]

**web-service** uses **app=nginx** as a Selector to logically group and target our three Pods. The three Pods' respective IP addresses are listed as the Service's endpoints. When a request reaches our Service, it will be served by one of the Pods listed in the **Endpoints** section as a result of the Service's load balancing mechanism.

`127.0.0.1` 和 `0.0.0.0` 是两种**本地地址**，但它们在网络服务监听和通信中有**非常不同的含义**：

---

### 🔹 `127.0.0.1`：本地回环地址（localhost）

- 表示**仅限本机访问**，不允许其他设备访问。
    
- 如果程序监听 `127.0.0.1:8080`：
    
    - **只能在本机访问**，例如浏览器打开 `http://127.0.0.1:8080` 是可以的；
        
    - 其他机器（甚至同一局域网内）**无法访问该端口**。
        

✔ 安全性高，但无法对外提供服务。

---

### 🔹 `0.0.0.0`：通配地址（wildcard）

- 表示**监听所有本地网络接口**，包括：
    
    - `127.0.0.1`（本地回环）
        
    - 本地 IP（如 `192.168.x.x` 或 `10.x.x.x`）
        
    - 公网 IP（如果存在）
        
- 如果程序监听 `0.0.0.0:8080`：
    
    - **本机访问没问题**（`http://127.0.0.1:8080`）
        
    - **其他设备也可以访问**（如 `http://192.168.1.10:8080`）
        

✔ 用于开发 Web 服务、API、Socket 服务器时对外暴露端口。

---

### 📊 总结对比表

|地址|含义|可被外部访问|用途场景|
|---|---|---|---|
|`127.0.0.1`|本机回环（localhost）|❌ 否|开发测试，仅本机访问|
|`0.0.0.0`|所有本地 IP 通配符|✅ 是|启动服务器，对局域网或公网开放|

---

### 🧪 示例说明

假设你在一台电脑上启动一个 HTTP 服务：

```bash
python -m http.server 8080
```

- 如果监听在 `127.0.0.1:8080`，你只能在本机浏览器访问 `http://127.0.0.1:8080`
    
- 如果监听在 `0.0.0.0:8080`，你可以在同一局域网的其他电脑访问该服务（如通过 `http://192.168.1.10:8080`）
    
对于使用 Docker 驱动程序的 Minikube 集群，由于 Docker 网络模型的限制，无法从主机工作站访问 NodePort。在这种情况下，另一种应用程序访问方式是通过 Minikube 隧道。此选项允许将服务 ClusterIP 直接作为外部 IP 暴露在主机上。首先通过 LoadBalancer 服务暴露 Web 服务器应用程序，然后启用隧道：


# Liveness, Readiness, and Startup Probes
虽然容器化应用程序被调度到集群各个节点的 Pod 中运行，但有时这些应用程序可能会无响应或在启动过程中出现延迟。实施 Liveness 和 Readiness 探针可以让 kubelet 控制 Pod 容器内运行应用程序的健康状况，并强制重启无响应的应用程序。

## Liveness
如果 Pod 中的某个容器已经成功运行了一段时间，但容器内运行的应用程序突然停止响应我们的请求，那么该容器对我们来说就不再有用了。这种情况可能由于应用程序死锁或内存压力等原因而发生。在这种情况下，建议重新启动该容器以使应用程序可用。

与其手动重新启动，不如使用 Liveness Probe。Liveness Probe 会检查应用程序的健康状况，如果健康检查失败，kubelet 会自动重启受影响的容器。

Liveness Probes can be set by defining:

- Liveness command
- Liveness HTTP request
- TCP Liveness probe
- gRPC Liveness probe

the kubelet sends the HTTP GET request to the /healthz endpoint of the application, on port 8080. If that returns a failure, then the kubelet will restart the affected container; otherwise, it would consider the application to be alive:
**...**  
     **livenessProbe:**  
       **httpGet:**  
         **path: /healthz**  
         **port: 8080**  
         **httpHeaders:**  
         **- name: X-Custom-Header**  
           **value: Awesome**  
       **initialDelaySeconds: 15**  
       **periodSeconds: 5**  
**...**


With [TCP Liveness Probe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-tcp-liveness-probe), the kubelet attempts to open the TCP Socket to the container running the application. If it succeeds, the application is considered healthy, otherwise the kubelet would mark it as unhealthy and restart the affected container.
**...**  
    **livenessProbe:**  
      **tcpSocket:**  
        **port: 8080**  
      **initialDelaySeconds: 15**  
      **periodSeconds: 5**  
**...**

The [gRPC Liveness Probe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-grpc-liveness-probe) can be used for applications implementing the [gRPC health checking protocol](https://github.com/grpc/grpc/blob/master/doc/health-checking.md). It requires for a **port** to be defined, and optionally a **service** field may help adapt the probe for liveness or readiness by allowing the use of the same port.

**...  
    livenessProbe:  
      grpc:  
        port: 2379  
      initialDelaySeconds: 10  
...**

## Readiness Probes
有时，应用程序在初始化时必须满足某些条件才能准备好处理流量。这些条件包括确保依赖服务已准备就绪，或者确认需要加载大型数据集等。在这种情况下，我们会使用就绪探针 (Readiness Probes)，等待特定条件发生。只有这样，应用程序才能处理流量。

A Pod with containers that do not report ready status will not receive traffic from Kubernetes Services.

**...**  
    **readinessProbe:**  
          **exec:**  
            **command:**  
            **- cat**  
            **- /tmp/healthy**  
          **initialDelaySeconds: 5**   
          **periodSeconds: 5**  
**...**

Readiness Probes are configured similarly to Liveness Probes. Their configuration fields and options also remain the same. Readiness probes are also defined as Readiness command, Readiness HTTP request, TCP readiness probe, and gRPC readiness probe.

## Startup Probes
探针家族的最新成员是启动探针。此探针专为可能需要更多时间才能完全初始化的旧版应用程序而设计，其目的是延迟存活探针和就绪探针，使其延迟足够长的时间，以便应用程序完全初始化。