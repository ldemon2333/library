要在 Kubernetes 中 **定义一个 Service 实现 Pod 之间的通信**，你可以使用一个 **`ClusterIP` 类型的 Service**（这是默认类型），它会在集群内暴露一个 DNS 名称，供其它 Pod 通过它通信。

---

## ✅ 场景示意

我们有多个副本（Pod），例如由 Deployment 创建。我们希望它们之间能够互相通过 `Service` 名称通信（例如进行 leader election、数据同步、任务分发等）。

---

## 📄 Service 定义示例（`leader-demo-service.yaml`）

```yaml
apiVersion: v1
kind: Service
metadata:
  name: leader-demo
spec:
  selector:
    app: leader-demo
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

### ✅ 含义：

- `name: leader-demo` → 这就是别的 Pod 使用的 DNS 名；
    
- `selector` 选择所有 `app=leader-demo` 的 Pod；
    
- 所有 Ready 的 Pod 都会加入这个服务；
    
- 默认创建的是 `ClusterIP` 服务，仅集群内可访问。
    

---

## 🔧 Deployment 配合（Pod 监听服务）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: leader-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: leader-demo
  template:
    metadata:
      labels:
        app: leader-demo
    spec:
      containers:
      - name: myapp
        image: your-image
        ports:
        - containerPort: 80
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
```

---

## 🧪 Pod 间通信演示：

假设容器内运行了一个 HTTP 服务器监听 80 端口：

- 在任意一个 Pod 内使用如下命令访问其他副本：
    

```bash
curl http://leader-demo:80/
```

- DNS 会轮询 Ready 的 Pod；
    

你也可以通过 SRV 记录访问所有副本的 IP 列表（更高级用法）：

```bash
dig SRV leader-demo.default.svc.cluster.local
```

---

## 🧠 补充说明：

- 所有 Pod 都通过 `leader-demo` 这个服务通信；
    
- 服务不会绑定某一个 Pod，而是动态指向当前 Ready 的所有 Pod；
    
- 如果你希望每个 Pod 都能被单独访问，可以使用 StatefulSet + Headless Service（`clusterIP: None`），我也可以提供这个例子。
    

---

## 📌 总结

|目的|方式|
|---|---|
|所有 Pod 互相通信|定义 `ClusterIP` 类型的 Service|
|单独访问某个 Pod|使用 StatefulSet + Headless Service|
|外部访问|定义 `NodePort` / `LoadBalancer`|

---

需要我给出一个完整的 **Pod 间通信 demo**，比如：

- Pod A 向 Pod B 发请求；
    
- 或者三个 Pod 通过 Service 向 Leader 发心跳？
    

告诉我语言（Go/Python/其他）和功能，我可以写代码。