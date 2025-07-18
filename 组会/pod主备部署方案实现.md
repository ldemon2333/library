下面是一个完整的 Kubernetes 多副本部署实例，其中：

- 使用 `ConfigMap` 实现简单的 Leader Election；
    
- 通过容器内部读取 `ConfigMap` 中是否是 “自己是 leader” 决定是否 Ready；
    
- 非 Leader 的副本不会被加入到 Service Endpoint 中，因此不会提供服务；
    
- 使用 `Deployment` 保证多副本部署。
    

---

## 🎯 示例场景：

部署一个 `nginx` 容器副本为 3 个，但只有一个能真正对外服务。Active 实例由 `ConfigMap` 中的 hostname 决定。

---

## 1️⃣ 创建 ConfigMap，记录当前 leader：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: leader-config
data:
  leader: ""
```

---

## 2️⃣ 容器脚本（写在镜像或 ConfigMap 中），检查是否是 leader：

`check-leader.sh`：

```bash
#!/bin/sh
POD_NAME=$(hostname)
LEADER=$(cat /config/leader)

if [ "$POD_NAME" = "$LEADER" ]; then
  exit 0
else
  exit 1
fi
```

---

## 3️⃣ Deployment 资源清单（leader 由手动设置或 Controller 设置）：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: single-active-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: single-active
  template:
    metadata:
      labels:
        app: single-active
    spec:
      containers:
      - name: nginx
        image: nginx
        readinessProbe:
          exec:
            command:
              - sh
              - -c
              - /check-leader.sh
          initialDelaySeconds: 5
          periodSeconds: 5
        volumeMounts:
        - name: config
          mountPath: /config
          readOnly: true
        - name: check-script
          mountPath: /check-leader.sh
          subPath: check-leader.sh
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: leader-config
      - name: check-script
        configMap:
          name: check-leader-script
          defaultMode: 0755
```

---

## 4️⃣ 将脚本放入一个 ConfigMap 中：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: check-leader-script
data:
  check-leader.sh: |
    #!/bin/sh
    POD_NAME=$(hostname)
    LEADER=$(cat /config/leader)

    if [ "$POD_NAME" = "$LEADER" ]; then
      exit 0
    else
      exit 1
    fi
```

---

## 5️⃣ 手动设置一个 Pod 为 leader：

找到某个 Pod 名称（例如 `single-active-nginx-xxxxxx`），执行：

```bash
kubectl get pods -l app=single-active
kubectl edit configmap leader-config
# 修改内容为：
# data:
#   leader: single-active-nginx-xxxxxx
```

---

## 6️⃣ 验证：

```bash
kubectl get endpoints
kubectl get pods -o wide
```

你会看到只有一个 Pod 出现在 Service 的 endpoint 中。

---

## 🚀 下一步：自动 Leader 选举

这个例子中 Leader 是手动设置的。若你想要 **自动选举**，建议用：

- 使用 `Lease` API + client-go 的 `leaderelection`；
    
- 或者用 `zookeeper/etcd` 做分布式锁选举；
    
- 我可以帮你写 Go 语言版基于 Lease 的完整自动选举实例（包含控制 Leader Ready 状态）。
    

是否需要？是否你希望语言为 Go 或 Python？可告知我平台和语言要求。


好的，下面是一个 **基于 Kubernetes Lease 资源实现自动 Leader Election 并结合 `readinessProbe` 控制流量只由 Leader Pod 提供** 的完整示例，使用 **Go + client-go 官方 LeaderElection 库** 实现。

---

# ✅ 自动 Leader Election + readinessProbe 示例（Go + Kubernetes Lease）

---

## 🧱 项目结构（简洁示意）：

```
leader-example/
├── main.go                # 主程序
├── Dockerfile             # 构建镜像
├── k8s/
│   ├── deployment.yaml    # 启动 3 个副本
│   └── rbac.yaml          # Lease 操作权限
```

---

## 1️⃣ Go 程序（main.go）

```go
package main

import (
    "context"
    "fmt"
    "os"
    "time"

    coordinationv1 "k8s.io/api/coordination/v1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/util/runtime"
    "k8s.io/apimachinery/pkg/util/wait"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/rest"
    "k8s.io/client-go/tools/leaderelection"
    "k8s.io/client-go/tools/leaderelection/resourcelock"
    "k8s.io/client-go/tools/record"
    "k8s.io/client-go/tools/record/util"
)

var readyFile = "/tmp/ready"

func main() {
    runtime.Must(wait.PollUntilContextTimeout(context.TODO(), time.Second, time.Minute, true, func(ctx context.Context) (done bool, err error) {
        _, err = rest.InClusterConfig()
        return err == nil, nil
    }))

    cfg, err := rest.InClusterConfig()
    if err != nil {
        panic(err)
    }

    client := kubernetes.NewForConfigOrDie(cfg)
    podName := os.Getenv("POD_NAME")
    podNamespace := os.Getenv("POD_NAMESPACE")

    rl, err := resourcelock.New(
        resourcelock.LeasesResourceLock,
        podNamespace,
        "example-leader-election",
        client.CoreV1(),
        client.CoordinationV1(),
        resourcelock.ResourceLockConfig{
            Identity: podName,
        },
    )
    if err != nil {
        panic(err)
    }

    electCtx, cancel := context.WithCancel(context.Background())

    go func() {
        leaderelection.RunOrDie(electCtx, leaderelection.LeaderElectionConfig{
            Lock:          rl,
            LeaseDuration: 15 * time.Second,
            RenewDeadline: 10 * time.Second,
            RetryPeriod:   2 * time.Second,
            Callbacks: leaderelection.LeaderCallbacks{
                OnStartedLeading: func(ctx context.Context) {
                    fmt.Println("I am the leader now:", podName)
                    os.WriteFile(readyFile, []byte("ok"), 0644)
                    <-ctx.Done()
                },
                OnStoppedLeading: func() {
                    fmt.Println("I am no longer the leader:", podName)
                    os.Remove(readyFile)
                },
                OnNewLeader: func(identity string) {
                    if identity != podName {
                        fmt.Println("New leader is:", identity)
                    }
                },
            },
            ReleaseOnCancel: true,
            Name:            "example-leader-election",
        })
    }()

    select {}
}
```

---

## 2️⃣ Dockerfile

```Dockerfile
FROM golang:1.22 as builder
WORKDIR /app
COPY main.go .
RUN go mod init leader && go mod tidy && go build -o /leader main.go

FROM gcr.io/distroless/static
COPY --from=builder /leader /leader
ENTRYPOINT ["/leader"]
```

---

## 3️⃣ readinessProbe（只让 leader 成为 ready）

```yaml
readinessProbe:
  exec:
    command: ["cat", "/tmp/ready"]
  initialDelaySeconds: 5
  periodSeconds: 3
```

---

## 4️⃣ k8s 部署文件（k8s/deployment.yaml）

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
      serviceAccountName: leader-election-sa
      containers:
      - name: leader
        image: your-registry/leader:latest
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        readinessProbe:
          exec:
            command: ["cat", "/tmp/ready"]
          initialDelaySeconds: 5
          periodSeconds: 3
```

---

## 5️⃣ RBAC 权限（k8s/rbac.yaml）

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: leader-election-sa

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: leader-election-role
rules:
- apiGroups: ["coordination.k8s.io"]
  resources: ["leases"]
  verbs: ["get", "watch", "list", "create", "update", "patch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: leader-election-binding
subjects:
- kind: ServiceAccount
  name: leader-election-sa
roleRef:
  kind: Role
  name: leader-election-role
  apiGroup: rbac.authorization.k8s.io
```

---

## ✅ 流程解释

1. 所有 Pod 启动后运行 leader election；
    
2. 只有一 Pod 获胜，写 `/tmp/ready`；
    
3. readinessProbe 检查此文件，仅该 Pod 成为 ready；
    
4. 只有该 Pod 会被加入 Service；
    
5. leader 退出或失效后，其他副本自动竞争并提供服务。
    

你的理解完全正确：**每个 Pod 的文件系统都是隔离的**，`/tmp/ready` 写入的是自己的容器内的文件，**不会影响其他 Pod**。这正是设计的一部分。

---

## 🧠 回顾目标：

我们希望：

- 所有 Pod 同时运行；
    
- 只有 **1 个 Pod 对外提供服务**；
    
- 使用 Kubernetes 的 `readinessProbe` 实现仅此 Pod 加入 Service Endpoints。
    

---

## ✅ 关键点解释：

### 🔁 Pod 如何竞争？

Pod 中运行的 Go 程序使用 Kubernetes 提供的 **Leader Election API（基于 Lease 对象）** 实现竞争。

**过程如下：**

1. 每个 Pod 启动后都使用 Kubernetes API 尝试成为 leader；
    
2. 所有 Pod 监听/尝试 `coordination.k8s.io/v1 Lease` 对象；
    
3. Kubernetes `client-go` 中的 `leaderelection.RunOrDie(...)` 会协调谁是 leader；
    
4. 一旦当选，调用 `OnStartedLeading` 回调。
    

→ `OnStartedLeading` 就是写入 `/tmp/ready` 的地方。

---

### 📁 `/tmp/ready` 是本地标记，不是跨 Pod 通信

- 这个文件仅对当前 Pod 的 `readinessProbe` 起作用；
    
- readinessProbe 会检测 `/tmp/ready` 是否存在；
    
- 存在：表示当前 Pod 是 Leader → Ready → 可被访问；
    
- 不存在：NotReady → 不会加入 Service。
    

### ⛔️ 文件确实不能跨 Pod 共享，但这正是需要的效果！

- 我们就是用本地状态文件来区分：**谁是 Leader，谁不是**；
    
- 每个 Pod 都通过自己的 client-go 实例感知选举状态；
    
- 不是通过共享 `/tmp/ready` 实现协作，而是通过共享 **Lease 对象** 协作。
    

---



## ✅ 为什么这样设计？

- `Lease` 对象是全局共享资源，由 API Server 负责协调；
    
- 每个 Pod 运行的代码逻辑相同；
    
- client-go 库会自动尝试获取租约，并执行回调；
    
- readinessProbe 与本地 Leader 状态解耦，避免外部组件依赖；
    
- 避免复杂状态共享机制（如 etcd/redis）。
    

---

![[Pasted image 20250702173109.png]]

## ✅ 最终结果

- 多副本 Pod 同时启动；
    
- Kubernetes 原生控制谁提供服务（仅 Ready 的 Pod）；
    
- Leader 失效自动切换；
    
- 无需额外负载均衡器或外部锁。
    
