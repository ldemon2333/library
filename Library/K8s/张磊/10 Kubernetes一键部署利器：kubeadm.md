在今天的分享中，我通过一个小案例，和你近距离体验了Kubernetes的使用方法。

可以看到，Kubernetes推荐的使用方式，是用一个YAML文件来描述你所要部署的API对象。然后，统一使用kubectl apply命令完成对这个对象的创建和更新操作。

而Kubernetes里“最小”的API对象是Pod。Pod可以等价为一个应用，所以，Pod可以由多个紧密协作的容器组成。

在Kubernetes中，我们经常会看到它通过一种API对象来管理另一种API对象，比如Deployment和Pod之间的关系；而由于Pod是“最小”的对象，所以它往往都是被其他对象控制的。这种组合方式，正是Kubernetes进行容器编排的重要模式。

而像这样的Kubernetes API对象，往往由Metadata和Spec两部分组成，其中Metadata里的Labels字段是Kubernetes过滤对象的主要手段。

在这些字段里面，容器想要使用的数据卷，也就是Volume，正是Pod的Spec字段的一部分。而Pod里的每个容器，则需要显式的声明自己要挂载哪个Volume。

上面这些基于YAML文件的容器管理方式，跟Docker、Mesos的使用习惯都是不一样的，而从docker run这样的命令行操作，向kubectl apply YAML文件这样的声明式API的转变，是每一个容器技术学习者，必须要跨过的第一道门槛。

所以，如果你想要快速熟悉Kubernetes，请按照下面的流程进行练习：

- 首先，在本地通过Docker测试代码，制作镜像；
- 然后，选择合适的Kubernetes API对象，编写对应YAML文件（比如，Pod，Deployment）；
- 最后，在Kubernetes上部署这个YAML文件。

更重要的是，在部署到Kubernetes之后，接下来的所有操作，要么通过kubectl来执行，要么通过修改YAML文件来实现，**就尽量不要再碰Docker的命令行了**。

在实际使用Kubernetes的过程中，相比于编写一个单独的Pod的YAML文件，我一定会推荐你使用一个replicas=1的Deployment。请问，这两者有什么区别呢？

这是一个非常常见也非常实用的问题。在 Kubernetes 中，**直接使用 Pod** 和 **使用 `Deployment`（即使是 `replicas: 1`）** 看似类似，实则在**可靠性、可维护性和扩展性**上有明显差异。

---

## ✅ 建议使用 `Deployment` 而非裸 Pod 的主要原因

|对比点|裸 Pod（直接定义 Pod）|Deployment（哪怕是 `replicas: 1`）|
|---|---|---|
|✅ **重建能力**|Pod 一旦崩溃不会自动恢复|Deployment 控制器会自动重建 Pod|
|✅ **声明式管理**|不支持滚动更新、版本控制|支持滚动升级、回滚、变更追踪|
|✅ **控制器管理**|没有控制器，靠手动管理|Deployment 会自动监控与调节状态|
|✅ **水平扩展**|无法扩展（除非复制 YAML）|一行改 `replicas: N` 即可扩展|
|✅ **与 CI/CD 集成**|不方便集成和发布管理|易与 GitOps、ArgoCD 等集成|
|✅ **可维护性**|运维压力大，不利于团队协作|Deployment 是社区推荐的最佳实践|

---

## 🧪 一个具体例子来对比

### 1. 裸 Pod 定义（不可恢复）

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: nginx
      image: nginx
```

> 如果你用 `kubectl delete pod mypod` 删除它，**它就没了，永远不回来**。

---

### 2. Deployment（自动恢复）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mydeployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: nginx
          image: nginx
```

> 如果你删除 Pod，Deployment 控制器会立刻自动重新创建一个新的。

---

## 🧠 为什么有些人还会用裸 Pod？

|场景|原因|
|---|---|
|临时调试/测试|例如：临时运行一个 `busybox`、`curl` 或 `debug` 容器|
|DaemonSet/StatefulSet 控制|某些场景不适合用 Deployment，比如每节点一个 Pod|

---

## ✅ 总结一句话

> 在正式环境中，**永远不要直接创建裸 Pod**，即使你只需要一个实例，也请使用 `Deployment` 或其他控制器（如 `StatefulSet`、`Job`）。

