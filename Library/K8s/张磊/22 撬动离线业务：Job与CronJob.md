我和你详细分享了Deployment、StatefulSet，以及DaemonSet这三个编排概念。你有没有发现它们的共同之处呢？

实际上，它们主要编排的对象，都是“在线业务”，即：Long Running Task（长作业）。比如，我在前面举例时常用的Nginx、Tomcat，以及MySQL等等。这些应用一旦运行起来，除非出错或者停止，它的容器进程会一直保持在Running状态。

但是，有一类作业显然不满足这样的条件，这就是“离线业务”，或者叫作Batch Job（计算业务）。这种业务在计算完成后就直接退出了，而此时如果你依然用Deployment来管理这种业务的话，就会发现Pod会在计算结束后退出，然后被Deployment Controller不断地重启；而像“滚动更新”这样的编排功能，更无从谈起了。

所以，早在Borg项目中，Google就已经对作业进行了分类处理，提出了LRS（Long Running Service）和Batch Jobs两种作业形态，对它们进行“分别管理”和“混合调度”。

首先，Job Controller控制的对象，直接就是Pod。

其次，Job Controller在控制循环中进行的调谐（Reconcile）操作，是根据实际在Running状态Pod的数目、已经成功退出的Pod的数目，以及parallelism、completions参数的值共同计算出在这个周期里，应该创建或者删除的Pod数目，然后调用Kubernetes API来执行这个操作。

# 总结
在今天这篇文章中，我主要和你分享了Job这个离线业务的编排方法，讲解了completions和parallelism字段的含义，以及Job Controller的执行原理。

紧接着，我通过实例和你分享了Job对象三种常见的使用方法。但是，根据我在社区和生产环境中的经验，大多数情况下用户还是更倾向于自己控制Job对象。所以，相比于这些固定的“模式”，掌握Job的API对象，和它各个字段的准确含义会更加重要。

最后，我还介绍了一种Job的控制器，叫作：CronJob。这也印证了我在前面的分享中所说的：用一个对象控制另一个对象，是Kubernetes编排的精髓所在。

