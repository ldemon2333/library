https://devopscube.com/learn-kubernetes-complete-roadmap/

**Service Discovery**: It is one of the key areas of Kubernetes. You need to have basic knowledge of client-side and server-side service discovery. To put it simply, in client-side service discovery, the **request goes to a service registry** to get the endpoints available for backend services. In server-side service discovery, the r**equest goes to a load balancer** and the load balancer uses the service registry to get the ending of backend services.


1. **Networking Basis**: Networking is a key part of Kubernetes. To understand Kubernetes networking, you need to have a fair knowledge of the following topics.
    1. CIDR Notation & Type of [IP Addresses](https://devopscube.com/ip-address-tutorial/)
    2. L2, L3, L4 & L7 Layers (OSI Layers)
    3. SSL/TLS: One way & Mutual TLS
    4. Proxy
    5. DNS
    6. IPVS/IPTables/NFtables
    7. Software Defined Networking (SDN)
    8. Virtual Interfaces
    9. Overlay networking

2. **Linux**: Although there is support for Windows, most of the Kubernetes implementations are based on Linux. Following are the key Linux concepts you should know.
    1. IPTables
    2. Filesystems
    3. Mount points 
    4. Swap
    5. Systemd
    6. journalctl, syslog
    7. SELinux
    8. AppArmor

![[Pasted image 20250610210731.png]]
1. **Cluster high availability:** Most organizations use managed Kubernetes services (GKE, EKS, AKS, etc). So the cloud provider takes care of the cluster's control plane's high availability. However, it is very important to learn the high availability concepts in scaling the cluster in multiple zones and regions. It will help you in real-time projects and devops interviews.
大多数组织使用托管的 Kubernetes 服务（GKE、EKS、AKS 等）。因此，云提供商负责集群控制平面的高可用性。然而，学习在多区域和多地域扩展集群时的高可用性概念非常重要。这将帮助你应对实时项目和 DevOps 面试。


2. **Network Design**: While it is easy to set up a cluster in an open network without restrictions, it is not that easy in a corporate network. As a DevOps engineer, you should understand the Kubernetes network design and requirements so that you can **collaborate with the network team** better. For example, When I was working with kubernetes setup on Google Cloud, we used a CIDR pod range that was not routable in the corporate network. As a workaround, we had to deploy IP masquerading for the pod network.


Following are my cluster setup suggestions.

1. [**Kubernetes the Hard Way:**](https://github.com/techiescamp/kubernetes-projects/tree/main/01-kubernetes-the-hard-way-aws?ref=devopscube.com) I would suggest you start with Kubernetes the Hard Way setup. It helps you understand all the configurations involved in bootstrapping a kubernetes cluster. If you want to **work on production clusters**, this lab will help you a lot. Also, it is ok if you are not able to setup the full fledged cluster. Atleast you will learn about all the components that make up the cluster.
2. [**Kubeadm Cluster Setup**](https://devopscube.com/setup-kubernetes-cluster-kubeadm/): Learning kubeadm cluster setup helps you in [Kubernetes certification](https://devopscube.com/best-kubernetes-certifications/) preparation. Also, it helps you automate Kubernetes cluster setup with best practices.
3. [**Minikube:**](https://devopscube.com/kubernetes-minikube-tutorial/) If you want to have a minimal development cluster setup, minikube is the best option.
4. [**Kind:**](https://kind.sigs.k8s.io/?ref=devopscube.com) Kind is another local development Kubernetes cluster setup.
5. [**Vagrant Automated Kubernetes**](https://github.com/techiescamp/vagrant-kubeadm-kubernetes?ref=devopscube.com)**:** If you prefer to have a multi-VM-based local Kubernetes cluster setup, you can try the automated vagrant setup that uses Kubeadm to bootstrap the cluster.

# Learn about Cluster Configurations
Once you have a working cluster, you should learn about the **key cluster configurations**. This knowledge will be particularly helpful when working in a self-hosted Kubernetes setup.

Even if you use a managed Kubernetes cluster for your project, there may be certain cluster configurations that you need to modify.

For example, if you set up a cluster in a hybrid network, you may need to configure it with an **on-premises private DNS** server for private DNS resolution. This can be done via CoreDNS configuration.

**Refer to the** [Kubernetes Cluster configurations](https://devopscube.com/kubernetes-cluster-configurations/) guide for more details.

Also, having a solid understanding of cluster configurations will help you with Kubernetes certifications (CKA & CKS) where you need to troubleshoot cluster misconfiguration and issues.


# Understand Kubeconfig File
**`Kubeconfig`** file is a YAML file that contains all the cluster information and credentials to connect to the cluster.

As a Devops Engineer, You should learn to connect to kubernetes clusters in different ways using the Kubeconfig file. Because you will be responsible for setting up cluster **authentication for CI/CD systems**, providing cluster access to developers, etc.

So spend some time, understanding the Kubeconfig file structure and associated parameters.

Check out the [complete Kubeconfig file guide](https://devopscube.com/kubernetes-kubeconfig-file/) to learn everything about the Kubeconfig file.

# Understand Kubernetes Objects And Resources
You will quite often come across the names "**Kubernetes Object**" and "**Kubernetes Resource**"

First, you need to Understand the difference between an object and a resource in kubernetes.

To put it simply, anything a **user creates and persists in Kubernetes is an object**. For example, a namespace, pod, Deployment configmap, Secret, etc.

Before creating an object, you represent it in a YAML or JSON format. It is called an **Object Specification (Spec)**. You declare the desired state of the object on the Object Spec. Once the object is created, you can retrieve its details from the Kubernetes API using Kubectl or client libraries.

As we discussed earlier in the prerequisite section, everything in Kubernetes is an API. To create different object types, there are **API endpoints provided by the Kubernetes API server**. Those **object-specific api-endpoints are called resources**. For example, an endpoint to create a pod is called a **pod resource**.

So when you try to create a Kubernetes Object using Kubectl, it converts the YAML spec to JSON format and sends it to the Pod resource (Pod API endpoint).

You can refer to the [Kubernetes objects vs resource](https://devopscube.com/kubernetes-objects-resources/) guide for more details.

# Learn About Pod & Associated Resources
Once you have an understanding of Kubernetes Objects and resources, you can start with a native Kubernetes object called Pod. A pod is a basic building block of Kubernetes.

You should learn all the **Pod concepts** and their associated objects like **Service, Ingress, Persistent Volume, Configmap, and Secret**. Once you know everything about a pod, it is very easy to learn other pod-dependent objects like deployments, Daemonset, etc.

First, learn about the Pod Resource Definition (YAML). A typical Pod YAML contains the following high-level constructs.

1. Kind
2. Metadata
3. Annotations
4. Labels
5. Selectors

Refer to [Kubernetes Pod Explained](https://devopscube.com/kubernetes-pod/) blog to learn all the basics about Pod.

Once you have a basic understanding of the above, move on to hands-on learning. These concepts will make more sense when you do hands-on.

Following are the **hands-on tasks** to learn about Pod and its associated objects.

1. Deploy a pod
2. Deploy pod on the specific worker node
3. Add service to pod
4. Expose the pod Service using Nodeport
5. Expose the Pod Service using Ingress
6. Setup Pod resources & limits
7. Setup Pod with startup, liveness, and readiness probes.
8. Add Persistent Volume to the pod.
9. Attach configmap to pod
10. Add Secret to pod
11. multi-container pods (sidecar container pattern)
12. [Init containers](https://devopscube.com/kubernetes-init-containers/)
13. Ephemeral containers
14. Static Pods
15. Learn to troubleshoot Pods

Few advanced pod scheduling concepts.

1. Pod Preemption & Priority
2. Pod Disruption Budget
3. Pod Placement Using a Node Selector
4. Pod Affinity and Anti-affinity
5. Container Life Cycle Hooks


# Learn Pod Dependent Objects
Now that you have a better understanding of Pod and independent kubernetes resources, you can start learning about objects that are dependent on the Pod object. While learning this, you will come across concepts like HPA (Horizontal Pod Autoscaling) and VPA (Verification Pod Autoscaling)

1. Replicaset
2. [Deployment](https://devopscube.com/kubernetes-deployment-tutorial/)
3. [Daemonsets](https://devopscube.com/kubernetes-daemonset/)
4. Statefulset
5. [Jobs & Cronjobs](https://devopscube.com/create-kubernetes-jobs-cron-jobs/)

# Learn Ingress & Ingress Controllers
To expose applications to the outside world or end users, kubernetes has a native object called ingress.

Many engineers get confused with Ingress due to less knowledge of Ingress controllers. Ensure you go through the concept of Ingress and Ingress controllers and understand it correctly. Because it is the base of exposing applications to the outside world.

You can start with the following comprehensive guides.

1. [Kubernetes Ingress Explained](https://devopscube.com/kubernetes-ingress-tutorial/)
2. [Setting up Nginx Ingress Controller](https://devopscube.com/setup-ingress-kubernetes-nginx-controller/)

Also, learn about the [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/?ref=devopscube.com). It provides advanced features over Ingress.

# Learn End-to-end Microservices Application Deployment on Kubernetes
Once you understand the basics of these objects, you can try deploying an end-to-end microservices application on Kubernetes. Start with simple use cases and gradually increase complexity.

I would suggest you get a domain name and try setting up a microservice application from scratch and host it on your domain.

You don't need to develop an application for this. Choose any open-source microservice-based application and deploy it. My suggestion is to choose the open-source [pet clinic microservice application](https://github.com/spring-petclinic/spring-petclinic-microservices?ref=devopscube.com) based on Spring Boot.

Following are the high-level tasks.

1. [Build Docker images](https://devopscube.com/build-docker-image/) for all the services. Ensure you optimize the Dockerfile to [reduce the Docker Image size](https://devopscube.com/reduce-docker-image-size/).
2. Create manifests for all the services. (Deployment, Statefulset, Services, Configmaps, Secrets, etc). Try to add all the supported parameters for each object type.
3. Expose the front end with service type ClusterIp
4. Deploy Nginx Ingress controller and expose it with service type Loadbalancer.
5. Create an Ingress object with the front-end as the backend service.
6. Map the load balancer IP to the domain name.
7. Create an ingress object with a DNS name with the backend as a front-end service name.
8. Validate the application.

# Learn About Securing Kubernetes Cluster
Security is a key aspect of Kubernetes. There are many ways to implement security best practices in Kubernetes starting from building a secure container image.

Following the native ways of implementing security in kubernetes.

1. Service account
2. Pod Security Context
3. Seccomp & AppArmor
4. Role Based Access Control (RBAC)
5. Attribute-based access control (ABAC)
6. Network Policies

The following are the open-source tools you need to look at.

1. Open Policy Agent
2. Kyverno
3. [Kube-bench](https://devopscube.com/kube-bench-guide/)
4. Kube-hunter
5. Falco

# Learn About Kubernetes Configuration Management Tools
Now that you have a good understanding of all Kubernetes objects and deploying applications on Kubernetes, you can start learning about Kubernetes configuration management tools.

When you start working on a real-time project in an organization, you will see the usage of configuration management tools to deploy applications on Kubernetes.

Because in organizations, there are different environments like dev, stage, pre-prod, and production. You cannot create individual YAML files for each environment and manage them manually. So you need a system to manage Kubernetes YAML configurations effectively.

Following are the popular and widely adopted Kubernetes tools to manage YAML.

1. [Helm](https://devopscube.com/install-configure-helm-kubernetes/) (Templating Engine)
2. [Kuztomize](https://devopscube.com/kustomize-tutorial/) (Overlay Engine)

# Learn About Kubernetes Opearator Pattern
Kubernetes Operators is an advanced concept.

To understand operators, first, you need to learn the following Kubernetes concepts.

1. Custom resource definitions
2. Admission controllers
3. Validating and mutating Webhooks

To get started with operators, you can try setting the following operators on Kubernetes.

1. Prometheus Operator
2. MySQL Operator

If you are a Go developer or you want to learn to extend/customize kubernetes, I would suggest you **create your own operator** using Golang.


# Learn Important K8s Configurations
While learning kubernetes, you might use a cluster in open network connectivity.

So most of the tasks get executed without any issues. However, this is not the case with clusters set up on corporate networks.

So, here are some custom cluster configurations you should be aware of

1. Custom DNS server
2. Custom image registry
3. Shipping logs to external logging systems
4. Kubernetes OpenID Connect
5. Segregating & securing Nodes for PCI & PII Workloads

# Learn K8s Best Practices
Following are the resources that might help and add value to the Kubernetes learning process in terms of best practices.

1. **12 Factor Apps**: It is a methodology that talks about how to code, deploy and maintain modern microservices-based applications. Since Kubernetes is a cloud-native microservices platform, it is a must-know concept for DevOps engineers. So when you work on a real-time kubernetes project, you can implement these 12-factor principles.
2. **Kubernetes Failure Stories**: Kubernetes failure stories is a website that has a list of articles that talk about failures in Kubernetes implementation. If you read those stories, you can avoid those mistakes in your kubernetes implementation.
3. **Case Studies From Organizations:** Spend time on use cases published by organizations on Kubernetes usage and scaling. You can learn a lot from them. Following are some of the case studies that are worth reading.
    1. [Scheduling 300,000 Kubernetes Pods in Production Daily](https://www.youtube.com/watch?v=wjy35HfIP_k&ref=devopscube.com)
    2. [Scaling Kubernetes to 7,500 Nodes](https://openai.com/blog/scaling-kubernetes-to-7500-nodes/?ref=devopscube.com)

# Real-World Kubernetes Case Studies
When I spoke to the DevOps community, I found that a common issue was the lack of real-world experience with Kubernetes. If you don't have an active Kubernetes project in your organization, you can refer to case studies and learning materials published by organizations that use Kubernetes. This will also help you in Kubernetes interviews.

Here are some good real-world Kubernetes case studies that can enhance your Kubernetes knowledge:

1. [List of Kubernetes User Case Studies](https://kubernetes.io/case-studies/?ref=devopscube.com)Official Case Studies
2. [How OpenAI Scaled Kubernetes to 7,500 Nodes](https://openai.com/blog/scaling-kubernetes-to-7500-nodes/?ref=devopscube.com)Blog
3. [Testing 500 Pods Per Node](https://cloud.redhat.com/blog/500_pods_per_node?ref=devopscube.com)Blog
4. [Dynamic Kubernetes Cluster Scaling at Airbnb](https://medium.com/airbnb-engineering/dynamic-kubernetes-cluster-scaling-at-airbnb-d79ae3afa132?ref=devopscube.com)Blog
5. [Scaling 100 to 10,000 pods on Amazon EKS](https://aws.amazon.com/blogs/containers/scale-from-100-to-10000-pods-on-amazon-eks?ref=devopscube.com)Blog