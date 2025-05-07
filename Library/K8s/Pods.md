_Pods_ are the smallest deployable units of computing that you can create and manage in Kubernetes.

A _Pod_ (as in a pod of whales or pea pod) is a group of one or more [containers](https://kubernetes.io/docs/concepts/containers/), with shared storage and network resources, and a specification for how to run the containers. A Pod's contents are always co-located and co-scheduled, and run in a shared context. A Pod models an application-specific "logical host": it contains one or more application containers which are relatively tightly coupled. In non-cloud contexts, applications executed on the same physical or virtual machine are analogous to cloud applications executed on the same logical host.

As well as application containers, a Pod can contain [init containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) that run during Pod startup. You can also inject [ephemeral containers](https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/) for debugging a running Pod.

# What is a Pod?
The shared context of a Pod is a set of Linux namespaces, cgroups, and potentially other facets of isolation - the same things that isolate a [container](https://kubernetes.io/docs/concepts/containers/). Within a Pod's context, the individual applications may have further sub-isolations applied.

A Pod is similar to a set of containers with shared namespaces and shared filesystem volumes.

Pods in a Kubernetes cluster are used in two main ways:

- **Pods that run a single container**. The "one-container-per-Pod" model is the most common Kubernetes use case; in this case, you can think of a Pod as a wrapper around a single container; Kubernetes manages Pods rather than managing the containers directly.
    
- **Pods that run multiple containers that need to work together**. A Pod can encapsulate an application composed of [multiple co-located containers](https://kubernetes.io/docs/concepts/workloads/pods/#how-pods-manage-multiple-containers) that are tightly coupled and need to share resources. These co-located containers form a single cohesive unit.
    
    Grouping multiple co-located and co-managed containers in a single Pod is a relatively advanced use case. You should use this pattern only in specific instances in which your containers are tightly coupled.
    
    You don't need to run multiple containers to provide replication (for resilience or capacity); if you need multiple replicas, see [Workload management](https://kubernetes.io/docs/concepts/workloads/controllers/).
    

# Using Pods

## Workload resources for managing pods
Usually you don't need to create Pods directly, even singleton Pods. Instead, create them using workload resources such as [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) or [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/). If your Pods need to track state, consider the [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) resource.

Each Pod is meant to run a single instance of a given application. If you want to scale your application horizontally (to provide more overall resources by running more instances), you should use multiple Pods, one for each instance. In Kubernetes, this is typically referred to as _replication_. Replicated Pods are usually created and managed as a group by a workload resource and its [controller](https://kubernetes.io/docs/concepts/architecture/controller/).

Pods natively provide two kinds of shared resources for their constituent containers: [networking](https://kubernetes.io/docs/concepts/workloads/pods/#pod-networking) and [storage](https://kubernetes.io/docs/concepts/workloads/pods/#pod-storage).

# Working with Pods
You'll rarely create individual Pods directly in Kubernetes—even singleton Pods. This is because Pods are designed as relatively ephemeral, disposable entities. When a Pod gets created (directly by you, or indirectly by a [controller](https://kubernetes.io/docs/concepts/architecture/controller/)), the new Pod is scheduled to run on a [Node](https://kubernetes.io/docs/concepts/architecture/nodes/) in your cluster. The Pod remains on that node until the Pod finishes execution, the Pod object is deleted, the Pod is _evicted_ for lack of resources, or the node fails.

#### Note:
Restarting a container in a Pod should not be confused with restarting a Pod. A Pod is not a process, but an environment for running container(s). A Pod persists until it is deleted.

The name of a Pod must be a valid [DNS subdomain](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#dns-subdomain-names) value, but this can produce unexpected results for the Pod hostname. For best compatibility, the name should follow the more restrictive rules for a [DNS label](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#dns-label-names).

## Pod OS

## Pods and controllers
You can use workload resources to create and manage multiple Pods for you. A controller for the resource handles replication and rollout and automatic healing in case of Pod failure. For example, if a Node fails, a controller notices that Pods on that Node have stopped working and creates a replacement Pod. The scheduler places the replacement Pod onto a healthy Node.

Here are some examples of workload resources that manage one or more Pods:


# Resource sharing and communication
Pods enable data sharing and communication among their constituent containers.

## Storage in Pods
A Pod can specify a set of shared storage [volumes](https://kubernetes.io/docs/concepts/storage/volumes/). All containers in the Pod can access the shared volumes, allowing those containers to share data. Volumes also allow persistent data in a Pod to survive in case one of the containers within needs to be restarted. See [Storage](https://kubernetes.io/docs/concepts/storage/) for more information on how Kubernetes implements shared storage and makes it available to Pods.

## Pod networking
Each Pod is assigned a unique IP address for each address family. Every container in a Pod shares the network namespace, including the IP address and network ports. Inside a Pod (and **only** then), the containers that belong to the Pod can communicate with one another using `localhost`. When containers in a Pod communicate with entities _outside the Pod_, they must coordinate how they use the shared network resources (such as ports). Within a Pod, containers share an IP address and port space, and can find each other via `localhost`. The containers in a Pod can also communicate with each other using standard inter-process communications like SystemV semaphores or POSIX shared memory. Containers in different Pods have distinct IP addresses and can not communicate by OS-level IPC without special configuration. Containers that want to interact with a container running in a different Pod can use IP networking to communicate.

Containers within the Pod see the system hostname as being the same as the configured `name` for the Pod. There's more about this in the [networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/) section.

