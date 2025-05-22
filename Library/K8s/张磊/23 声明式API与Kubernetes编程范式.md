**那么，到底什么才是“声明式API”呢？**

答案是，kubectl apply命令。

可是，它跟kubectl replace命令有什么本质区别吗？

实际上，你可以简单地理解为，kubectl replace的执行过程，是使用新的YAML文件中的API对象，**替换原有的API对象**；而kubectl apply，则是执行了一个**对原有API对象的PATCH操作**。

>类似地，kubectl set image和kubectl edit也是对已有API对象的修改。

更进一步地，这意味着kube-apiserver在响应命令式请求（比如，kubectl replace）的时候，一次只能处理一个写请求，否则会有产生冲突的可能。而对于声明式请求（比如，kubectl apply），**一次能处理多个写操作，并且具备Merge能力**。

Istio项目，实际上就是一个基于Kubernetes项目的微服务治理框架。它的架构非常清晰，如下所示：
![[Pasted image 20250507213732.png]]

在上面这个架构图中，我们不难看到Istio项目架构的核心所在。**Istio最根本的组件，是运行在每一个应用Pod里的Envoy容器**。

这个Envoy项目是Lyft公司推出的一个高性能C++网络代理，也是Lyft公司对Istio项目的唯一贡献。

而Istio项目，则把这个代理服务以sidecar容器的方式，运行在了每一个被治理的应用Pod中。我们知道，Pod里的所有容器都共享同一个Network Namespace。所以，Envoy容器就能够通过配置Pod里的iptables规则，把整个Pod的进出流量接管下来。

这时候，Istio的控制层（Control Plane）里的Pilot组件，就能够通过调用每个Envoy容器的API，对这个Envoy代理进行配置，从而实现微服务治理。

我们一起来看一个例子。

假设这个Istio架构图左边的Pod是已经在运行的应用，而右边的Pod则是我们刚刚上线的应用的新版本。这时候，Pilot通过调节这两Pod里的Envoy容器的配置，从而将90%的流量分配给旧版本的应用，将10%的流量分配给新版本应用，并且，还可以在后续的过程中随时调整。这样，一个典型的“灰度发布”的场景就完成了。比如，Istio可以调节这个流量从90%-10%，改到80%-20%，再到50%-50%，最后到0%-100%，就完成了这个灰度发布的过程。

更重要的是，在整个微服务治理的过程中，无论是对Envoy容器的部署，还是像上面这样对Envoy代理的配置，用户和应用都是完全“无感”的。

这时候，你可能会有所疑惑：Istio项目明明需要在每个Pod里安装一个Envoy容器，又怎么能做到“无感”的呢？

实际上，**Istio项目使用的，是Kubernetes中的一个非常重要的功能，叫作Dynamic Admission Control。**

在Kubernetes项目中，当一个Pod或者任何一个API对象被提交给APIServer之后，总有一些“初始化”性质的工作需要在它们被Kubernetes项目正式处理之前进行。比如，自动为所有Pod加上某些标签（Labels）。

而这个“初始化”操作的实现，借助的是一个叫作Admission的功能。它其实是Kubernetes项目里一组被称为Admission Controller的代码，可以选择性地被编译进APIServer中，在API对象创建之后会被立刻调用到。

但这就意味着，如果你现在想要添加一些自己的规则到Admission Controller，就会比较困难。因为，这要求重新编译并重启APIServer。显然，这种使用方法对Istio来说，影响太大了。

所以，Kubernetes项目为我们额外提供了一种“热插拔”式的Admission机制，它就是Dynamic Admission Control，也叫作：Initializer。

**Istio项目的核心，就是由无数个运行在应用Pod中的Envoy容器组成的服务代理网格**。这也正是Service Mesh的含义。

而这个机制得以实现的原理，正是借助了Kubernetes能够对API对象进行在线更新的能力，这也正是**Kubernetes“声明式API”的独特之处：**

- 首先，所谓“声明式”，指的就是我只需要提交一个定义好的API对象来“声明”，我所期望的状态是什么样子。
- 其次，“声明式API”允许有多个API写端，以PATCH的方式对API对象进行修改，而无需关心本地原始YAML文件的内容。
- 最后，也是最重要的，有了上述两个能力，Kubernetes项目才可以基于对API对象的增、删、改、查，在完全无需外界干预的情况下，完成对“实际状态”和“期望状态”的调谐（Reconcile）过程。

所以说，**声明式API，才是Kubernetes项目编排能力“赖以生存”的核心所在**，希望你能够认真理解。

此外，不难看到，无论是对sidecar容器的巧妙设计，还是对Initializer的合理利用，Istio项目的设计与实现，其实都依托于Kubernetes的声明式API和它所提供的各种编排能力。可以说，Istio是在Kubernetes项目使用上的一位“集大成者”。

而在使用Initializer的流程中，最核心的步骤，莫过于Initializer“自定义控制器”的编写过程。它遵循的，正是标准的“Kubernetes编程范式”，即：

> **如何使用控制器模式，同Kubernetes里API对象的“增、删、改、查”进行协作，进而完成用户业务逻辑的编写过程。**

# 总结
在今天这篇文章中，我为你重点讲解了Kubernetes声明式API的含义。并且，通过对Istio项目的剖析，我为你说明了它使用Kubernetes的Initializer特性，完成Envoy容器“自动注入”的原理。

事实上，从“使用Kubernetes部署代码”，到“使用Kubernetes编写代码”的蜕变过程，正是你从一个Kubernetes用户，到Kubernetes玩家的晋级之路。

而，如何理解“Kubernetes编程范式”，如何为Kubernetes添加自定义API对象，编写自定义控制器，正是这个晋级过程中的关键点，也是我要在后面几篇文章中分享的核心内容。

