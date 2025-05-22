在本篇，我们就先从一种特殊的Volume开始，来帮助你更加深入地理解Pod对象各个重要字段的含义。

这种特殊的Volume，叫作Projected Volume，你可以把它翻译为“投射数据卷”。

在Kubernetes中，有几种特殊的Volume，它们存在的意义不是为了存放容器里的数据，也不是用来进行容器和宿主机之间的数据交换。这些特殊Volume的作用，是为容器提供预先定义好的数据。所以，从容器的角度来看，这些Volume里的信息就是仿佛是**被Kubernetes“投射”（Project）进入容器当中的**。这正是Projected Volume的含义。

到目前为止，Kubernetes支持的Projected Volume一共有四种：

1. Secret；
    
2. ConfigMap；
    
3. Downward API；
    
4. ServiceAccountToken。

在今天这篇文章中，我首先和你分享的是Secret。它的作用，是帮你把Pod想要访问的加密数据，存放到Etcd中。然后，你就可以通过在Pod的容器里挂载Volume的方式，访问到这些Secret里保存的信息了。

Secret最典型的使用场景，莫过于存放数据库的Credential信息，比如下面这个例子：

![[Pasted image 20250507153927.png]]
接下来，我们再来看Pod的另一个重要的配置：容器健康检查和恢复机制。

在Kubernetes中，你可以为Pod里的容器定义一个健康检查“探针”（Probe）。这样，kubelet就会根据这个Probe的返回值决定这个容器的状态，而不是直接以容器镜像是否运行（来自Docker返回的信息）作为依据。这种机制，是生产环境中保证应用健康存活的重要手段。

我们一起来看一个Kubernetes文档中的例子。

![[Pasted image 20250507154217.png]]
在这个Pod中，我们定义了一个有趣的容器。它在启动之后做的第一件事，就是在/tmp目录下创建了一个healthy文件，以此作为自己已经正常运行的标志。而30 s过后，它会把这个文件删除掉。

与此同时，我们定义了一个这样的livenessProbe（健康检查）。它的类型是exec，这意味着，它会在容器启动后，在容器里面执行一条我们指定的命令，比如：“cat /tmp/healthy”。这时，如果这个文件存在，这条命令的返回值就是0，Pod就会认为这个容器不仅已经启动，而且是健康的。这个健康检查，在容器启动5 s后开始执行（initialDelaySeconds: 5），每5 s执行一次（periodSeconds: 5）。

而30 s之后，我们再查看一下Pod的Events：
```
$ kubectl describe pod test-liveness-exec
```
你会发现，这个Pod在Events报告了一个异常：
```
FirstSeen LastSeen Count From SubobjectPath Type Reason Message --------- -------- ----- ---- ------------- -------- ------ ------- 2s 2s 1 {kubelet worker0} spec.containers{liveness} Warning Unhealthy Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
```
显然，这个健康检查探查到/tmp/healthy已经不存在了，所以它报告容器是不健康的。那么接下来会发生什么呢？

我们不妨再次查看一下这个Pod的状态：
![[Pasted image 20250507154408.png]]
这时我们发现，Pod并没有进入Failed状态，而是保持了Running状态。这是为什么呢？

其实，如果你注意到RESTARTS字段从0到1的变化，就明白原因了：这个异常的容器已经被Kubernetes重启了。在这个过程中，Pod保持Running状态不变。

需要注意的是：Kubernetes中并没有Docker的Stop语义。所以虽然是Restart（重启），但实际却是重新创建了容器。

这个功能就是Kubernetes里的**Pod恢复机制**，也叫restartPolicy。它是Pod的Spec部分的一个标准字段（pod.spec.restartPolicy），默认值是Always，即：任何时候这个容器发生了异常，它一定会被重新创建。

但一定要强调的是，Pod的恢复过程，永远都是发生在当前节点上，而不会跑到别的节点上去。事实上，一旦一个Pod与一个节点（Node）绑定，除非这个绑定发生了变化（pod.spec.node字段被修改），否则它永远都不会离开这个节点。这也就意味着，如果这个宿主机宕机了，这个Pod也不会主动迁移到其他节点上去。

而如果你想让Pod出现在其他的可用节点上，就必须使用Deployment这样的“控制器”来管理Pod，哪怕你只需要一个Pod副本。这就是我在第12篇文章《牛刀小试：我的第一个容器化应用》最后给你留的思考题的答案，即一个单Pod的Deployment与一个Pod最主要的区别。

而作为用户，你还可以通过设置restartPolicy，改变Pod的恢复策略。除了Always，它还有OnFailure和Never两种情况：

- Always：在任何情况下，只要容器不在运行状态，就自动重启容器；
- OnFailure: 只在容器 异常时才自动重启容器；
- Never: 从来不重启容器。

在实际使用时，我们需要根据应用运行的特性，合理设置这三种恢复策略。

比如，一个Pod，它只计算1+1=2，计算完成输出结果后退出，变成Succeeded状态。这时，你如果再用restartPolicy=Always强制重启这个Pod的容器，就没有任何意义了。

而如果你要关心这个容器退出后的上下文环境，比如容器退出后的日志、文件和目录，就需要将restartPolicy设置为Never。因为一旦容器被自动重新创建，这些内容就有可能丢失掉了（被垃圾回收了）。

值得一提的是，Kubernetes的官方文档，把restartPolicy和Pod里容器的状态，以及Pod状态的对应关系，[总结了非常复杂的一大堆情况](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#example-states)。实际上，你根本不需要死记硬背这些对应关系，只要记住如下两个基本的设计原理即可：

1. **只要Pod的restartPolicy指定的策略允许重启异常的容器（比如：Always），那么这个Pod就会保持Running状态，并进行容器重启**。否则，Pod就会进入Failed状态 。
    
2. **对于包含多个容器的Pod，只有它里面所有的容器都进入异常状态后，Pod才会进入Failed状态**。在此之前，Pod都是Running状态。此时，Pod的READY字段会显示正常容器的个数，比如：

```
$ kubectl get pod test-liveness-exec 
NAME READY STATUS RESTARTS AGE 
liveness-exec 0/1 Running 1 1m
```

所以，假如一个Pod里只有一个容器，然后这个容器异常退出了。那么，只有当restartPolicy=Never时，这个Pod才会进入Failed状态。而其他情况下，由于Kubernetes都可以重启这个容器，所以Pod的状态保持Running不变。

而如果这个Pod有多个容器，仅有一个容器异常退出，它就始终保持Running状态，哪怕即使restartPolicy=Never。只有当所有容器也异常退出之后，这个Pod才会进入Failed状态。



