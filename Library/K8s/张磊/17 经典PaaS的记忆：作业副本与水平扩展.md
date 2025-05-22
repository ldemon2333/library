
Deployment看似简单，但实际上，它实现了Kubernetes项目中一个非常重要的功能：Pod的“水平扩展/收缩”（horizontal scaling out/in）。这个功能，是从PaaS时代开始，一个平台级项目就必须具备的编排能力。

举个例子，如果你更新了Deployment的Pod模板（比如，修改了容器的镜像），那么Deployment就需要遵循一种叫作“滚动更新”（rolling update）的方式，来升级现有的容器。

而这个能力的实现，依赖的是Kubernetes项目中的一个非常重要的概念（API对象）：ReplicaSet。

ReplicaSet的结构非常简单，我们可以通过这个YAML文件查看一下：
![[Pasted image 20250507103711.png]]

从这个YAML文件中，我们可以看到，**一个ReplicaSet对象，其实就是由副本数目的定义和一个Pod模板组成的**。不难发现，它的定义其实是Deployment的一个子集。

**更重要的是，Deployment控制器实际操纵的，正是这样的ReplicaSet对象，而不是Pod对象。**

还记不记得我在上一篇文章《编排其实很简单：谈谈“控制器”模型》中曾经提出过这样一个问题：对于一个Deployment所管理的Pod，它的ownerReference是谁？

所以，这个问题的答案就是：ReplicaSet。

明白了这个原理，我再来和你一起分析一个如下所示的Deployment：

![[Pasted image 20250507103754.png]]

可以看到，这就是一个我们常用的nginx-deployment，它定义的Pod副本个数是3（spec.replicas=3）。

那么，在具体的实现上，这个Deployment，与ReplicaSet，以及Pod的关系是怎样的呢？

我们可以用一张图把它描述出来：
![[Pasted image 20250507103817.png]]
通过这张图，我们就很清楚地看到，一个定义了replicas=3的Deployment，与它的ReplicaSet，以及Pod的关系，实际上是一种“层层控制”的关系。

其中，ReplicaSet负责通过“控制器模式”，保证系统中Pod的个数永远等于指定的个数（比如，3个）。这也正是Deployment只允许容器的restartPolicy=Always的主要原因：只有在容器能保证自己始终是Running状态的前提下，ReplicaSet调整Pod的个数才有意义。

而在此基础上，Deployment同样通过“控制器模式”，来操作ReplicaSet的个数和属性，进而实现“水平扩展/收缩”和“滚动更新”这两个编排动作。

其中，“水平扩展/收缩”非常容易实现，Deployment Controller只需要修改它所控制的ReplicaSet的Pod副本个数就可以了。

比如，把这个值从3改成4，那么Deployment所对应的ReplicaSet，就会根据修改后的值自动创建一个新的Pod。这就是“水平扩展”了；“水平收缩”则反之。

而用户想要执行这个操作的指令也非常简单，就是kubectl scale，比如：

```
kubectl scale deployment nginx-deployment --replicas=4
```
那么，“滚动更新”又是什么意思，是如何实现的呢？

接下来，我还以这个Deployment为例，来为你讲解“滚动更新”的过程。

首先，我们来创建这个nginx-deployment：
```
kubectl create -f nginx-deploument.yaml --record
```
注意，在这里，我额外加了一个–record参数。它的作用，是记录下你每次操作所执行的命令，以方便后面查看。

然后，我们来检查一下nginx-deployment创建后的状态信息：
![[Pasted image 20250507104117.png]]
在返回结果中，我们可以看到四个状态字段，它们的含义如下所示。

1. DESIRED：用户期望的Pod副本个数（spec.replicas的值）；
    
2. CURRENT：当前处于Running状态的Pod的个数；
    
3. UP-TO-DATE：当前处于最新版本的Pod的个数，所谓最新版本指的是Pod的Spec部分与Deployment里Pod模板里定义的完全一致；
    
4. AVAILABLE：当前已经可用的Pod的个数，即：既是Running状态，又是最新版本，并且已经处于Ready（健康检查正确）状态的Pod的个数。

可以看到，只有这个AVAILABLE字段，描述的才是用户所期望的最终状态。

而Kubernetes项目还为我们提供了一条指令，让我们可以实时查看Deployment对象的状态变化。这个指令就是kubectl rollout status：
![[Pasted image 20250507104204.png]]
在这个返回结果中，“2 out of 3 new replicas have been updated”意味着已经有2个Pod进入了UP-TO-DATE状态。

继续等待一会儿，我们就能看到这个Deployment的3个Pod，就进入到了AVAILABLE状态：
![[Pasted image 20250507104228.png]]
此时，你可以尝试查看一下这个Deployment所控制的ReplicaSet：
![[Pasted image 20250507104243.png]]
如上所示，在用户提交了一个Deployment对象后，Deployment Controller就会立即创建一个Pod副本个数为3的ReplicaSet。这个ReplicaSet的名字，则是由Deployment的名字和一个随机字符串共同组成。

如上所示，在用户提交了一个Deployment对象后，Deployment Controller就会立即创建一个Pod副本个数为3的ReplicaSet。这个ReplicaSet的名字，则是由Deployment的名字和一个随机字符串共同组成。

这个随机字符串叫作pod-template-hash，在我们这个例子里就是：3167673210。ReplicaSet会把这个随机字符串加在它所控制的所有Pod的标签里，从而保证这些Pod不会与集群里的其他Pod混淆。

而ReplicaSet的DESIRED、CURRENT和READY字段的含义，和Deployment中是一致的。所以，**相比之下，Deployment只是在ReplicaSet的基础上，添加了UP-TO-DATE这个跟版本有关的状态字段。**

这个时候，如果我们修改了Deployment的Pod模板，“滚动更新”就会被自动触发。

修改Deployment有很多方法。比如，我可以直接使用kubectl edit指令编辑Etcd里的API对象。

![[Pasted image 20250507104410.png]]

这个kubectl edit指令，会帮你直接打开nginx-deployment的API对象。然后，你就可以修改这里的Pod模板部分了。比如，在这里，我将nginx镜像的版本升级到了1.9.1。

> 备注：kubectl edit并不神秘，它不过是把API对象的内容下载到了本地文件，让你修改完成后再提交上去。

kubectl edit指令编辑完成后，保存退出，Kubernetes就会立刻触发“滚动更新”的过程。你还可以通过kubectl rollout status指令查看nginx-deployment的状态变化：

这时，你可以通过查看Deployment的Events，看到这个“滚动更新”的流程：

![[Pasted image 20250507104618.png]]

可以看到，首先，当你修改了Deployment里的Pod定义之后，Deployment Controller会使用这个修改后的Pod模板，创建一个新的ReplicaSet（hash=1764197365），这个新的ReplicaSet的初始Pod副本数是：0。

然后，在Age=24 s的位置，Deployment Controller开始将这个新的ReplicaSet所控制的Pod副本数从0个变成1个，即：“水平扩展”出一个副本。

紧接着，在Age=22 s的位置，Deployment Controller又将旧的ReplicaSet（hash=3167673210）所控制的旧Pod副本数减少一个，即：“水平收缩”成两个副本。

如此交替进行，新ReplicaSet管理的Pod副本数，从0个变成1个，再变成2个，最后变成3个。而旧的ReplicaSet管理的Pod副本数则从3个变成2个，再变成1个，最后变成0个。这样，就完成了这一组Pod的版本升级过程。

像这样，**将一个集群中正在运行的多个Pod版本，交替地逐一升级的过程，就是“滚动更新”。**

在这个“滚动更新”过程完成之后，你可以查看一下新、旧两个ReplicaSet的最终状态：

![[Pasted image 20250507104747.png]]

其中，旧ReplicaSet（hash=3167673210）已经被“水平收缩”成了0个副本。

**这种“滚动更新”的好处是显而易见的。**

比如，在升级刚开始的时候，集群里只有1个新版本的Pod。如果这时，新版本Pod有问题启动不起来，那么“滚动更新”就会停止，从而允许开发和运维人员介入。而在这个过程中，由于应用本身还有两个旧版本的Pod在线，所以服务并不会受到太大的影响。

现在我们可以扩展一下Deployment、ReplicaSet和Pod的关系图了。
![[Pasted image 20250507155148.png]]
如上所示，Deployment的控制器，实际上控制的是ReplicaSet的数目，以及每个ReplicaSet的属性。

而一个应用的版本，对应的正是一个ReplicaSet；这个版本应用的Pod数量，则由ReplicaSet通过它自己的控制器（ReplicaSet Controller）来保证。

通过这样的多个ReplicaSet对象，Kubernetes项目就实现了对多个“应用版本”的描述。

而明白了“应用版本和ReplicaSet一一对应”的设计思想之后，我就可以为你讲解一下Deployment对应用进行版本控制的具体原理了。

**然后，我们就可以在kubectl rollout undo命令行最后，加上要回滚到的指定版本的版本号，就可以回滚到指定版本了**。这个指令的用法如下：

# 总结
Deployment实际上是一个**两层控制器**。首先，它通过**ReplicaSet的个数**来描述应用的版本；然后，它再通过**ReplicaSet的属性**（比如replicas的值），来保证Pod的副本数量。

Deployment控制ReplicaSet（版本），ReplicaSet控制Pod（副本数）。这个两层控制关系一定要牢记。

不过，相信你也能够感受到，Kubernetes项目对Deployment的设计，实际上是代替我们完成了对“应用”的抽象，使得我们可以使用这个Deployment对象来描述应用，使用kubectl rollout命令控制应用的版本。

可是，在实际使用场景中，应用发布的流程往往千差万别，也可能有很多的定制化需求。比如，我的应用可能有会话黏连（session sticky），这就意味着“滚动更新”的时候，哪个Pod能下线，是不能随便选择的。

这种场景，光靠Deployment自己就很难应对了。对于这种需求，我在专栏后续文章中重点介绍的“自定义控制器”，就可以帮我们实现一个功能更加强大的Deployment Controller。


# 思考
你听说过金丝雀发布（Canary Deployment）和蓝绿发布（Blue-Green Deployment）吗？你能说出它们是什么意思吗？

当然可以，这两种部署策略都是用于**降低线上发布风险**、提高**服务可用性**的常见做法。以下是它们的定义和主要区别：

---

### 🟡 金丝雀发布（Canary Deployment）

**定义**：  
金丝雀发布是指将新版本的应用程序先部署给一小部分用户使用，观察运行状况和用户反馈。如果没有问题，再逐步扩大流量，最终替代旧版本。

**名字来源**：来自矿工带金丝雀下矿井探测毒气的做法，金丝雀先接触风险环境。

**特点**：

- 部署初期只影响少量用户。
    
- 可以逐步增加新版本的流量权重（10%→25%→50%→100%）。
    
- 通常配合灰度发布、A/B测试、流量网关（如 Istio、NGINX）等使用。
    
- 容易快速回滚。
    

**适合场景**：

- 大型用户系统；
    
- 不确定性较高的新功能上线；
    
- 希望平滑过渡并实时监控异常。
    

---

### 🔵🟢 蓝绿发布（Blue-Green Deployment）

**定义**：  
蓝绿发布是指准备两个完全独立的环境（一个“蓝”，一个“绿”），其中一个为线上运行环境（如蓝色），另一个为待发布的新版本（绿色）。当绿色部署完成并通过验证后，通过负载均衡器一键切换全部流量到绿色环境。

**特点**：

- 零停机部署；
    
- 切换快；
    
- 出现问题可以迅速切回旧版本；
    
- 需要额外资源维护两个完整环境。
    

**适合场景**：

- 资源充足；
    
- 对停机敏感的系统；
    
- 版本之间兼容性较高。
    

---

### ✅ 总结对比：

|特性|金丝雀发布|蓝绿发布|
|---|---|---|
|用户影响范围|逐步扩大（先小部分）|一次性全部切换|
|回滚策略|降低流量或回滚版本|切换回原环境|
|系统要求|流量调度机制|需完整的双环境|
|适用场景|高频更新，风险控制|零停机、系统容错要求高|

