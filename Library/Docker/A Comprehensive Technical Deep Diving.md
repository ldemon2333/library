https://medium.com/@furkan.turkal/how-does-docker-actually-work-the-hard-way-a-technical-deep-diving-c5b8ea2f0422

In this article, we will find answers the following questions:
- How has containerization changed the world?
- How the containers took over the cloud ecosystem?
- What does “container” mean exactly?
- What is “Docker” actually?

It abstracts the underlying infrastructure and provides a standardized way to package and deploy software.

部署困难和无法响应新功能的旧结构要求开发人员寻找新的服务设计。此时微服务应运而生。微服务架构实际上是一个分解成较小部分的整体结构。所有服务都独立工作。微服务彼此独立运行，它们可以单独更新、利用和扩展。这为开发和运营团队提供了机会，使他们能够保持软件最新，快速部署产品。使用这种结构的公司数量日益增多。最常见的用户是 Netflix、Google、Amazon 等。

微服务有积极的一面，但另一方面，随着小型服务和业务数量的增加，工作量也随之增加，很难像单个软件一样控制和合并它们。让它们平稳运行并降低硬件成本要困难得多。这一点，运营团队的职责是最大限度地提高自动化程度，以防止人为错误或故障，并开发可扩展的软件。

Linux container technologies allow developers to run multiple microservices on the same machine while not prepare a different environment to each service. Also, make them isolated them from each other by using containers.

Read the [Learning Containers From The Bottom Up](https://iximiuz.com/en/posts/container-learning-path/) article for containers learning path.

# 3. Docker Architecture

## 3.1 High-level Architecture
Docker’s higher level of [architecture](https://docs.docker.com/get-started/overview/#docker-architecture) revolves around a client-server model, where the client interacts with the [Docker daemon](https://docs.docker.com/config/daemon/) (server) to manage containers and related resources. At its core, Docker consists of three key components: the client, the daemon, and images. Client and daemon communicate using a REST API, over UNIX sockets or a network interface.
![[Pasted image 20250111000659.png]]
- [**Docker Client**](https://docs.docker.com/get-started/overview/#the-docker-client): is a command-line tool, API, or graphical interface that users interact with to issue commands and manage Docker resources. The client sends requests to the Docker daemon, which orchestrates the execution of those commands.
- [**Docker Daemon**](https://docs.docker.com/get-started/overview/#the-docker-daemon)**:** also known as Docker Engine, is a background service and long-running process that runs on the host machine and actually does the work of running and managing both containers and container images. The Docker daemon is responsible for managing the lifecycle of containers and orchestrating their operations. It listens for requests from the Docker client, manages containers, and coordinates various Docker operations. The daemon interacts with the host operating system’s kernel and leverages kernel features and modules for containerization, networking, and storage.
- [**Docker Desktop**](https://docs.docker.com/desktop/): is an easy-to-install application for your Mac, Windows or Linux environment that enables you to build and share containerized applications and microservices. With [Docker Extensions](https://docs.docker.com/desktop/extensions/), you can use third-party tools within Docker Desktop to extend its functionality.
- [**Docker Registry**](https://docs.docker.com/get-started/overview/#docker-registries): is a _registry_ that stores container images. [Docker Hub](https://hub.docker.com/) is a public registry that anyone can use, and Docker is configured to look for images on Docker Hub by default.

## 3.2 Low-level Architecture
![[Pasted image 20250111001223.png]]
1. Docker Desktop employs a _virtual machine (VM)_ to provide the necessary Linux environment.
2. Within the VM, the Docker client interacts with the Docker daemon _(dockerd)_ through a [RESTful API](https://docs.docker.com/engine/api/).
3. It uses containerd as its container runtime under the hood. Containerd is an industry-standard runtime that manages the lifecycle of a container on a physical or virtual machine. It is a daemon process that creates, starts, stops, and destroys containers.
4. [Containerd Plugins](https://github.com/containerd/containerd/blob/main/docs/PLUGINS.md) can be added to containerd to extend its functionality.
5. When a container is launched, the [shim (runtime v2) API](https://github.com/containerd/containerd/blob/main/runtime/v2/README.md#runtime-v2), which is part of containerd, acts as an intermediary between containerd and the _OCI runtime_.
6. The OCI runtime is responsible for setting up the container’s namespaces, control groups (cgroups), capabilities, and other settings required for containerization. It leverages the Linux kernel’s capabilities to isolate and control resources, enforce security, and manage the container’s behavior. Cgroups, short for control groups, are a Linux kernel feature that allows fine-grained resource allocation and control for processes. Capabilities within the Linux kernel provide privileges to containers, defining their permissions and access to various system resources.

## Docker CLI

[Docker CLI](https://docs.docker.com/engine/reference/commandline/cli/) tool is a command line application used to interact with the dockerd daemon. It includes several useful features. It handles standard UNIX-style arguments, and in many cases, it offers both short and long forms.

You can find all the [commands](https://github.com/docker/cli/tree/master/cli/command) in the repository if you are deeply interested.
![[Pasted image 20250111001749.png]]
A state machine provides a good summary of how the various states of the container are converted [[1]](https://www.tiejiang.org/23394.html)

## Docker Registry
Docker Registry is a service that allows users to store and distribute the container images. It serves as a centralized or decentralized locations where users can push their images, making them publicly available, accessible to other team members or systems for deployment.

## BuildKit


## Alternative Ways to Build Container Images
You do not need _just_ “Docker” to build container images. There are [many ways](https://twitter.com/developerguyba/status/1457792151710961667?s=20) for building OCI compliant container images:


## Container Runtime Interface (CRI)
Kubernetes is responsible for orchestration, runtime and knows how to run and check the status of _containers_. Docker launched [Swarm](https://docs.docker.com/engine/swarm), its own Kubernetes alternative, offering orchestration as a built-in Docker “mode”.

Kubernetes have a component called “kubelet” — an agent that runs on every node (physical machine) in a Kubernetes cluster. Container runtimes are a foundational component of a modern [containerized architecture](https://www.aquasec.com/cloud-native-academy/container-security/containerized-architecture/).

The Container Runtime Interface (CRI) allows Kubernetes to use any CRI-compliant runtime. Every Docker image can run in every container runtime.

There are various container runtimes. Each has special features and decide on trade-offs they make between performance, security and functionality.

Kubernetes is [deprecating Docker](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md#deprecation) as a container runtime after v1.20. [Don’t Panic:](https://kubernetes.io/blog/2020/12/02/dont-panic-kubernetes-and-docker/) Docker as an underlying runtime is being deprecated in favor of runtimes that use the Container Runtime Interface (CRI) created for Kubernetes. Switching to Containerd as the container runtime eliminates the middleman.

### dockershim
Dockershim implements CRI support for Docker .In the past, Kubernetes included a bridge called [dockershim](https://github.com/kubernetes/kubernetes/tree/master/pkg/kubelet/dockershim), which enabled Docker to work with CRI. From v1.20 onwards, dockershim will not be maintained, meaning that Docker is now [deprecated](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/2221-remove-dockershim) in Kubernetes. Kubernetes currently plans to remove support for Docker entirely in a future version, probably v1.22. [[15]](https://www.aquasec.com/cloud-native-academy/container-security/container-runtime-interface/)

## Containerd
[containerd](https://www.docker.com/blog/what-is-containerd-runtime/) is an industry-standard container runtime with an emphasis on simplicity, robustness and portability. containerd can manage the complete container lifecycle of its host system: image transfer and storage, container execution and supervision, low-level storage and network attachments, etc.

containerd is designed to be embedded into a larger system, rather than being used directly by developers or end-users.

containerd was designed to be used by Docker Daemon; and extracted its container runtime out into a new project.

Containers will schedule directly by containerd, which means they are not visible to Docker.
![[Pasted image 20250111102820.png]]

![[Pasted image 20250111103205.png]]

# Deep Diving into Components
As you may noticed already, Docker is not only component that doing every magic under the hood.

## Container Image
A container image is an immutable — meaning unchangeable, static file that includes executable code, so it can be deployed consistently and run an isolated process on any environment. It’s notion of a parent-child relationship and image layering. It can be seen as archives with a filesystem inside.

A container image is a combination of a JSON manifest and individual file-system layers, built onto a parent or base image. These layers encourage reuse of various components and configurations, so the user does not recreate everything from scratch. Constructing layers in an optimal manner can help reduce container size and improve performance.

Container images seems like a bit class/object concept. Image is like a class or template and then you can create any number of instances of that template and it has [OCI Runtime Specification](https://github.com/opencontainers/runtime-spec). It’s a definition of the standard container.

A container image rely on open standards and can be tagged or left untagged and only searchable through a true identifier.
![[Pasted image 20250111103828.png]]
Technically, you do not need the images to run containers! Interested in how? We will cover this in the further sections.

## Storage
[containers/storage](https://github.com/containers/storage) is a Go library which aims to provide methods for storing and managing three types of items: layers, images, and containers.

Docker image consists of several layers. Each layer corresponds to certain instructions in your [Dockerfile](https://docs.docker.com/engine/reference/builder/#format).


## Dockerfile
Initiating a new Dockerfile from scratch is easy with [docker init](https://docs.docker.com/engine/reference/commandline/init/) command.

The following above overview is a visualized version of a minimal Dockerfile:
![[Pasted image 20250111104214.png]]

[Layer](https://docs.docker.com/storage/storagedriver/#images-and-layers), is a [copy-on-write (CoW)](https://en.wikipedia.org/wiki/Copy-on-write) filesystem. [Each layer](https://github.com/containers/storage/blob/0cdedf0cbb50675bd5b8bccdc806e63b711d1e27/layers.go#L43-L126) is only a set of differences from the layer before it. Any layer can be stacked on top of each other. Both _adding_, and _removing_ files will result in a [new layer](https://github.com/containers/storage/blob/0cdedf0cbb50675bd5b8bccdc806e63b711d1e27/layers.go#L686-L827).

[Image](https://docs.docker.com/storage/storagedriver/#images-and-layers), is built up from a series of layers. Each layer represents an instruction in your Dockerfile. Multiple images can reference the same layer. Each layer except the very last one is read-only. Images are stateless.

[Container](https://docs.docker.com/storage/storagedriver/#container-and-layers), is a [read-write (RW)](https://en.wikipedia.org/wiki/Read%E2%80%93write_memory) layer. Child of an image’s top wriable layer. All writes to the container that add new or modify existing data are stored in this writable layer. When the container is deleted, the writable layer is also deleted. The underlying image remains unchanged. Each container has its own writable container layer. Multiple containers can be derived from a single image. All changes (such as writing new files, modifying existing files, and deleting files) are stored in this thin writable container layer, multiple containers can share access to the same underlying image and yet have their own data state.

[Container Layer](https://docs.docker.com/storage/storagedriver/#container-and-layers), Docker uses storage drivers to manage the contents of the image layers and the writable container layer. Each storage driver handles the implementation differently, but all drivers use stackable image layers and the [copy-on-write (CoW)](https://en.wikipedia.org/wiki/Copy-on-write) strategy.

Layers are stored similar to images internally. Each layer has its own directory in `/var/lib/docker/<driver>/layerdb/<algorithm>`. Docker stores all caches in `/var/lib/docker/<driver>`, where `<driver>` is the storage driver `overlay2` again. Read more [here](https://dominikbraun.io/blog/docker/docker-images-and-their-layers-explained/).

Total amount of data (on disk) that is used for the writable layer of each container is `size`, whereas `virtual size` is the amount of data used for the read-only image data used by the container plus the container’s writable layer `size`.


## Docker Manifest
[Docker Manifest](https://docs.docker.com/registry/spec/manifest-v2-2/), is a JSON-represented file that describes the image and provides metadata such as tags, a digital signature to verify the origin of the image, and documentation. Manifest is meant for consumption by a container runtime. [[4]](https://stackoverflow.com/a/47023753/5685796)

Manifest Lists (aka “fat manifest”), are defined in the [v2.2 image specification](https://github.com/docker/distribution/blob/master/docs/spec/manifest-v2-2.md) and exist mainly for the purpose of supporting _multi-architecture_ and _multi-platform_ images within an image registry. There are several [IANA](https://www.iana.org/assignments/media-types/media-types.xhtml) [Media Types](https://docs.docker.com/registry/spec/manifest-v2-2/#media-types) that Docker currently supported. If you want to view, create, and push the new manifests list object types, [manifest-tool](https://github.com/estesp/manifest-tool) is the tool you are looking for.

Although _dive_ says we achieved almost `%95+` efficiency score, the total size of the image is `316 MB`. This is because of unnecessary dependencies of `golang:1.17.3-alpine3.14` image. We can use [multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/) to optimize the image.


## Containers
A Linux container is nothing more than simply isolated and restricted L̵i̵n̵u̵x̵ ̵p̵̵̵r̵̵̵o̵̵̵c̵̵̵e̵̵̵s̵̵̵s̵̵̵e̵̵̵s̵̵̵ **boxes for running one or more processes** inside that runs on Linux. [[6]](https://iximiuz.com/en/posts/oci-containers/) What we mean by _box_ here that the _isolated_ process for start has its own process namespace. A containerized process interact with the kernel through _system calls_ and needs permissions just the same way that a regular process does. A container is a self contained execution environment that shares the kernel of the host system and which is (optionally) isolated from other containers in the system. A bare minimum container is just a single executable file inside.

If we shell into a container, we would see just the processes running inside that container it has its own process namespace. Typically the way that containers are used by one process per container. Container process tied in with the lifecycle of the container itself: starting/ending a container, starts/kills the container process. Entire lifecycle is tightly coupled.

With containers, it’s expected that all the dependencies. Pretty much above the kernel are packaged inside of the container. Containers share the host’s kernel. Within a container, you do not see the host’s entire filesystem; instead, you see a subset as the root directory is changed when the container is created. When you run the container in a OS, you actually do not install anyhing. It sits above the OS and it’s own world.

### Namespaces
The Linux kernel has a concept to common functionality and makes it available to many applications running on the system. This concept is called namespaces. It’s a form the foundation of container technology.

Namespaces allow containers to have isolation on 6 _+ 1_ levels. The purpose of each namespace is to wrap a particular global system resource in an abstraction that makes it appear to the processes within the namespace that they have their own isolated instance of the global resource.
![[Pasted image 20250111105622.png]]
**USER:** (USER): [CLONE_NEWUSER]: isolates user and group IDs. A process’s UserID and GroupID can be different inside and outside a user namespace

**PID:** (Process ID): [CLONE_NEWPID]: isolates process IDs, allow each container to have its own `[init](https://en.wikipedia.org/wiki/Init)` . A process has two PIDs: inside the namespace, and outside the namespace on the host system. Better check [An Init System inside the Docker Container](https://medium.com/@BeNitinAgarwal/an-init-system-inside-the-docker-container-3821ee233f4b) by [@BeNitinAgarwal](https://medium.com/@BeNitinAgarwal)

**UTS:** (UNIX Time-sharing System): [CLONE_NEWUTS]: isolates the hostname ([uname](https://man7.org/linux/man-pages/man2/uname.2.html)() syscall) and the [Network Information Service (NIS)](https://www.geeksforgeeks.org/linux-network-information-service/) domain name

**NET:** (Network): [CLONE_NEWNET]: isolates the network interface controllers (i.e., devices, IP addresses, IP routing tables, /proc/net directory, port numbers, etc.)

**MNT:** (Mount): [CLONE_NEWNS]: isolates filesystem mount points ([mount()](http://man7.org/linux/man-pages/man2/mount.2.html) and [umount()](http://man7.org/linux/man-pages/man2/umount.2.html) syscalls)

**IPC:** (Inter-process Communication): [CLONE_NEWIPC]: isolates [System V IPC](http://www.kernel.org/doc/man-pages/online/pages/man7/svipc.7.html) objects, [POSIX message queues](http://www.kernel.org/doc/man-pages/online/pages/man7/mq_overview.7.html)

[Namespace](https://man7.org/linux/man-pages/man7/namespaces.7.html) isolation means that groups of processes are separated such that they cannot “see” resources in other groups: [[11]](https://blog.ramlot.eu/containers/)
- Processes in different UTS namespaces see their dedicated hostname and can edit their hostname independently.
- Processes in different User namespaces see a their dedicates list of users and can add or remove users without affecting other processes.
- Processes get a different PID for each PID namespaces they are part of, each PID namespace has its own PID tree.
- Process is always in exactly one namespace of each type.

By putting a process in a namespace, you can restrict the resources that are visible to that process. The origin of namespaces date back to the [Plan 9](http://doc.cat-v.org/plan_9/4th_edition/papers/names).

### Control Groups (cgroups)
Cgroups allowes us to apply certain degree of isolation and restrict what a process is able to do: capabilitiees, resource limits, etc. It’s a fundamental notion of the runtime definition of a container. When you start a container, the runtime creates new cgroups for it.

Cgroup is not only for imposing limitation on CPU and memory usage; it also limits accesses to device files such as `/dev/sda1`.

[**Linux Capabilities**](https://man7.org/linux/man-pages/man7/capabilities.7.html) are used to allow binaries (executed by non-root users) to perform privileged operations without providing them all root permissions. Capabilities can be assigned to a processes/thread to determine wheter that processes/thread can perform certain actions. [@wifisecguy](https://medium.com/@wifisecguy) have a great series: “Understanding Linux Capabilities Series” [I](https://blog.pentesteracademy.com/linux-security-understanding-linux-capabilities-series-part-i-4034cf8a7f09), [II](https://blog.pentesteracademy.com/linux-security-understanding-linux-capabilities-series-part-ii-bccc08ce0973), [III](https://blog.pentesteracademy.com/linux-security-understanding-linux-capabilities-series-part-iii-c3fe47a03fe4)

[**lscgroup**](https://linux.die.net/man/1/lscgroup) (in [libcgroup-tools](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/resource_management_guide/chap-using_libcgroup_tools) package) provides obtaining information about controllers, control groups, and their parameters.

[**capsh**](https://man7.org/linux/man-pages/man1/capsh.1.html) provides a handy wrapper for certain types of capability testing and environment creation and some debugging features useful for summarizing capability state.

To limit total number of processes allowed within a control group, there is a control group called _pid,_ which can be prevent the effectiveness of a [fork bomb](https://en.wikipedia.org/wiki/Fork_bomb).

The Linux Kernel communicates information about cgroups through a set of pseudo-filesystems that typically reside at `/sys/fs/cgroup`. From inside the container, the list of its own cgroups is available from the `[/proc](https://man7.org/linux/man-pages/man5/proc.5.html)`. Each cgroup has an interface file called `cgroup.procs` that lists the PIDs of all processes belonging to the cgroup, one per line. [[12]](https://facebookmicrosites.github.io/cgroup2/docs/create-cgroups.html)


