StatefulSet其实就是对现有典型运维业务的容器化抽象。也就是说，你一定有方法在不使用Kubernetes、甚至不使用容器的情况下，自己DIY一个类似的方案出来。但是，一旦涉及到升级、版本管理等更工程化的能力，Kubernetes的好处，才会更加凸现。

比如，如何对StatefulSet进行“滚动更新”（rolling update）？

很简单。你只要修改StatefulSet的Pod模板，就会自动触发“滚动更新”:

在这里，我使用了kubectl patch命令。它的意思是，以“补丁”的方式（JSON格式的）修改一个API对象的指定字段，也就是我在后面指定的“spec/template/spec/containers/0/image”。

这样，StatefulSet Controller就会按照与Pod编号相反的顺序，从最后一个Pod开始，逐一更新这个StatefulSet管理的每个Pod。而如果更新发生了错误，这次“滚动更新”就会停止。此外，StatefulSet的“滚动更新”还允许我们进行更精细的控制，比如金丝雀发布（Canary Deploy）或者灰度发布，**这意味着应用的多个实例中被指定的一部分不会被更新到最新的版本**。

这个字段，正是StatefulSet的spec.updateStrategy.rollingUpdate的partition字段。

比如，现在我将前面这个StatefulSet的partition字段设置为2：

其中，kubectl patch命令后面的参数（JSON格式的），就是partition字段在API对象里的路径。所以，上述操作等同于直接使用 kubectl edit命令，打开这个对象，把partition字段修改为2。

这样，我就指定了当Pod模板发生变化的时候，比如MySQL镜像更新到5.7.23，那么只有序号大于或者等于2的Pod会被更新到这个版本。并且，如果你删除或者重启了序号小于2的Pod，等它再次启动后，也会保持原先的5.7.2版本，绝不会被升级到5.7.23版本。

DaemonSet。

顾名思义，DaemonSet的主要作用，是让你在Kubernetes集群里，运行一个Daemon Pod。 所以，这个Pod有如下三个特征：

1. 这个Pod运行在Kubernetes集群里的每一个节点（Node）上；
    
2. 每个节点上只有一个这样的Pod实例；
    
3. 当有新的节点加入Kubernetes集群后，该Pod会自动地在新节点上被创建出来；而当旧节点被删除后，它上面的Pod也相应地会被回收掉。

这个机制听起来很简单，但Daemon Pod的意义确实是非常重要的。我随便给你列举几个例子：

1. 各种网络插件的Agent组件，都必须运行在每一个节点上，用来处理这个节点上的容器网络；
    
2. 各种存储插件的Agent组件，也必须运行在每一个节点上，用来在这个节点上挂载远程存储目录，操作容器的Volume目录；
    
3. 各种监控组件和日志组件，也必须运行在每一个节点上，负责这个节点上的监控信息和日志搜集。

更重要的是，跟其他编排对象不一样，DaemonSet开始运行的时机，很多时候比整个Kubernetes集群出现的时机都要早。

这个乍一听起来可能有点儿奇怪。但其实你来想一下：如果这个DaemonSet正是一个网络插件的Agent组件呢？

这个时候，整个Kubernetes集群里还没有可用的容器网络，所有Worker节点的状态都是NotReady（NetworkReady=false）。这种情况下，普通的Pod肯定不能运行在这个集群上。所以，这也就意味着DaemonSet的设计，必须要有某种“过人之处”才行。

为了弄清楚DaemonSet的工作原理，我们还是按照老规矩，先从它的API对象的定义说起。

那么，**DaemonSet又是如何保证每个Node上有且只有一个被管理的Pod呢？**

显然，这是一个典型的“控制器模型”能够处理的问题。

DaemonSet Controller，首先从Etcd里获取所有的Node列表，然后遍历所有的Node。这时，它就可以很容易地去检查，当前这个Node上是不是有一个携带了name=fluentd-elasticsearch标签的Pod在运行。

而检查的结果，可能有这么三种情况：

1. 没有这种Pod，那么就意味着要在这个Node上创建这样一个Pod；
    
2. 有这种Pod，但是数量大于1，那就说明要把多余的Pod从这个Node上删除掉；
    
3. 正好只有一个这种Pod，那说明这个节点是正常的。
    

其中，删除节点（Node）上多余的Pod非常简单，直接调用Kubernetes API就可以了。

但是，**如何在指定的Node上创建新Pod呢？**

如果你已经熟悉了Pod API对象的话，那一定可以立刻说出答案：用nodeSelector，选择Node的名字即可。

![[Pasted image 20250507171548.png]]
所以，**我们的DaemonSet Controller会在创建Pod的时候，自动在这个Pod的API对象里，加上这样一个nodeAffinity定义**。其中，需要绑定的节点名字，正是当前正在遍历的这个Node。

# 总结
相比于Deployment，DaemonSet只管理Pod对象，然后通过nodeAffinity和Toleration这两个调度器的小功能，保证了每个节点上有且只有一个Pod。这个控制器的实现原理简单易懂，希望你能够快速掌握。

与此同时，DaemonSet使用ControllerRevision，来保存和管理自己对应的“版本”。这种“面向API对象”的设计思路，大大简化了控制器本身的逻辑，也正是Kubernetes项目“声明式API”的优势所在。

而且，相信聪明的你此时已经想到了，StatefulSet也是直接控制Pod对象的，那么它是不是也在使用ControllerRevision进行版本管理呢？

没错。在Kubernetes项目里，ControllerRevision其实是一个通用的版本管理对象。这样，Kubernetes项目就巧妙地避免了每种控制器都要维护一套冗余的代码和逻辑的问题。

