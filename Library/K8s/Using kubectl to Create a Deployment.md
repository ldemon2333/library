一旦拥有正在运行的 Kubernetes 集群，您就可以在其上部署容器化应用程序。为此，您需要创建一个 Kubernetes Deployment。部署指示 Kubernetes 如何创建和更新应用程序的实例。创建部署后，Kubernetes 控制平面会安排该部署中包含的应用程序实例在集群中的各个节点上运行。

Once the application instances are created, a Kubernetes Deployment controller continuously monitors those instances. If the Node hosting an instance goes down or is deleted, the Deployment controller replaces the instance with an instance on another Node in the cluster. **This provides a self-healing mechanism to address machine failure or maintenance.**

In a pre-orchestration world, installation scripts would often be used to start applications, but they did not allow recovery from machine failure. By both creating your application instances and keeping them running across Nodes, Kubernetes Deployments provide a fundamentally different approach to application management.

When you create a Deployment, you'll need to specify the container image for your application and the number of replicas that you want to run. You can change that information later by updating your Deployment;

