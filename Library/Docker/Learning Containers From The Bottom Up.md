- 容器可以做什么以及不可以做什么；
- 哪些是使用容器的最佳实践以及哪些不是；
- 哪些东西放在容器是安全的以及哪些不是。

# Containers Learning Path
I find the following learning order particularly helpful:
- Linux Containers: learn how-level implementation details.
- Container Images
- Container Managers
- Container Orchestrators: learn how Kubernetes coordinates containers in clusters
- Non-Linux Containers

# 容器不是虚拟机
容器是一种隔离（命名空间）且受约束（通过 cgroups、capabilities、seccomp）的进程。
![[Pasted image 20250113095704.png]]

To start a processes on Linux, you need to fork/exec it. But to start a containerized process, you need to create namespaces, configure cgroups, etc. Or, in other words, prepare a _box_ for a process to run inside. A **_Container Runtime_** is a special kind of (rather lower-level) software to create such boxes. A typical container runtime knows how to prepare the box and then how to start a containerized process in it. And since [most of the runtimes adhere to a common specification](https://iximiuz.com/en/posts/oci-containers/), containers become a standard unit of workload.

The most widely used container runtime out there is _runc_. And since _runc_ is a regular command-line tool, one can use it directly without Docker or any other higher-level containerization software!
![[Pasted image 20250113100051.png]]
_How runc starts a containerized process._

I got so fascinated by this finding that I even wrote a [whole series of articles on _container runtime shims_](https://iximiuz.com/en/tags/?tag=container-runtime-shim). A _shim_ is a piece of software that resides in between [a lower-level container runtime such as _runc_ and a higher-level container manager such as _containerd_](https://iximiuz.com/en/posts/journey-from-containerization-to-orchestration-and-beyond/). Since a shim needs to understand all the runtime's quirks really well, the series starts from the [in-depth analysis of the most widely used container runtime](https://iximiuz.com/en/posts/implementing-container-runtime-shim/).
![[Pasted image 20250113101938.png]]
_Container runtime shim in the wild._

# 允许容器不一定需要镜像
不过，构建镜像需要容器。

For folks familiar with how _runc_ starts containers, it's clear that images aren't really a part of the equation. Instead, to run a container, a runtime needs a so-called _bundle_ that consists of:
- a `config.json` file holding container parameters (path to an executable, env vars, etc.)
- a folder with the said executable and supporting files (if any).
![[Pasted image 20250113102209.png]]
Oftentimes, a bundle folder contains a file structure resembling a typical Linux distribution (`/var`, `/usr`, `/lib`, `/etc`, ...). When runc launches a container with such a bundle, the process inside gets a root filesystem that looks pretty much like your favorite Linux flavor, be it Debian, CentOS, or Alpine.

But such a file structure is not mandatory! So-called _scratch_ or _distroless_ containers are getting more and more popular nowadays, in particular, because [slimmer containers leave fewer chances for security vulnerabilities to sneak in](https://iximiuz.com/en/posts/thick-container-vulnerabilities/).

Check out [this article where I show how to create a container that has just a single Go binary inside](https://iximiuz.com/en/posts/not-every-container-has-an-operating-system-inside/):
![[Pasted image 20250113102433.png]]
_Inspecting a scratch image with [`dive`](https://github.com/wagoodman/dive)._

**_So, why do we need container images if they are not required to run containers?_**

When every container gets a multi-megabyte copy of the root filesystem, the space requirements increase drastically. So, images are there to solve the storage and distribution problems efficiently. Intrigued? [Read here how](https://iximiuz.com/en/posts/you-dont-need-an-image-to-run-a-container/).

**_Have you ever wondered how images are created?_**

The workflow popularized by Docker tries to make you think that images are primary and that containers are secondary. With `docker run <image>` you need an image to run a container. However, we already know that technically it's not true. But there is more! In actuality, you have to run (temporary and ephemeral) containers to build an image! [Read here why](https://iximiuz.com/en/posts/you-need-containers-to-build-an-image/).

# Single-Host Container Managers
Containers are meant to increase resource utilization of our servers.

A typical server now runs tens or hundreds of containers. Thus, they need to coexist efficiently. If a container runtime focuses on a single container lifecycle, **a Container Manager focuses on making many containers happily coexist on a single host.**

The main responsibilities of a container manager include pulling images, unpacking bundles, [configuring the inter-container networking](https://iximiuz.com/en/posts/container-networking-is-simple/), storing container logs, etc.

At this point, you probably think that Docker is a good example of such a manager. However, I find **_containerd_** a much more representative candidate. Much like runc, _containerd_ started as a component of Docker but then was extracted into a standalone project. Under the hood, _containerd_ can use _runc_ or any other runtime that implements the _containerd-shim_ interface. And the coolest part about it is that with _containerd_ you can run containers almost as easily as with Docker itself.

Check out this [post on how to use containerd from the command line](https://iximiuz.com/en/posts/containerd-command-line-clients/) - it's a great hands-on exercise bringing you closer to the actual containers.

If you want to know more about container manager internals, check out this [article showing how to implement a container manager from scratch](https://iximiuz.com/en/posts/conman-the-container-manager-inception/):

## containerd vs. docker
Finally, we are ready to understand Docker! If we omit the (now deprecated) Swarm thingy, Docker can be seen as follows:
- `dockerd` - a higher-level daemon sitting in front of the `containerd` daemon
- `docker` - a famous command-line client to interact with `dockerd`.
![Layered Docker architecture: docker (cli) -> dockerd -> containerd -> containerd-shim -> runc](https://iximiuz.com/container-learning-path/docker-containerd-runc-2000-opt.png)
From my point of view, Docker's main task currently is to make the container workflows developer-friendly. To make developer's life easier, Docker combines in one piece of software all main container use cases:

- build/pull/push/scan images
- launch/pause/inspect/kill containers
- [create networks/forward ports](https://iximiuz.com/en/posts/multiple-containers-same-port-reverse-proxy/)
- mount/unmount/remove volumes
- etc.

...but by 2021, almost every such use case got written a tailored piece of software (_podman_, _buildah_, _skopeo_, _kaniko_, etc.) to provide an alternative, and often better, solution.

# Multi-host Container Orchestrators
Coordinating containers running on a single host is hard. But coordinating containers across multiple hosts is exponentially harder. Remember Docker Swarm? Docker was already quite monstrous when the multi-host container orchestration was added, bringing one more responsibility for the existing daemon...

Omitting the issue with the bloated daemon, Docker Swarm seemed nice. But another orchestrator won the competition - Kubernetes! So, since ca. 2020, Docker Swarm is either obsolete or in maintenance mode, and we're all learning a couple of Ancient Greek words per week.

Kubernetes joins multiple servers (nodes) into a coordinated cluster, and every node gets a local agent called _kubelet_. Among other things, _kubelet_ is responsible for launching [_Pods_ (coherent groups of containers)](https://iximiuz.com/en/posts/containers-vs-pods/). But _kubelet_ doesn't do it on its own. Historically, it used `dockerd` for that purpose, but now this approach is deprecated in favor of a more generic _Container Runtime Interface (CRI)_.
![[Pasted image 20250113103640.png]]

_Kubernetes can use `containerd`, `cri-o`, or any other CRI-compatible runtime._

There is a lot of tasks for the container orchestrator.
- [How to group containers (or rather Pods) into higher-level primitives (ReplicaSets, Deployments, etc.)?](https://iximiuz.com/en/posts/kubernetes-vs-virtual-machines/)
- How to interconnect nodes running containers into a common network?
- [How to provide service discovery?](https://iximiuz.com/en/posts/service-discovery-in-kubernetes/)
- et cetera, et cetera...

Kubernetes, and other orchestrators like Nomad or AWS ECS, enabled teams to create isolated services more easily. It helped to solve lots of administrative problems, especially for bigger companies. But it created lots of new tech problems that didn't exist on the traditional VM-based setups! Managing lots of distributed services turned out to be quite challenging. [And that's how a Zoo of Cloud Native projects came to be.](https://iximiuz.com/en/posts/making-sense-out-of-cloud-native-buzz/)

# Some Containers are Virtual Machines
Now, when you have a decent understanding of containers - from both the implementation and usage standpoints - it's time to tell you the truth. **Containers aren't Linux processes!**

Even Linux containers aren't technically processes. They are rather **isolated and restricted environments _to run one or many processes inside_**.

Following the above definition, it shouldn't be surprising that at least some containers can be implemented using mechanisms other than namespaces and cgroups. And indeed, there are projects like Kata containers that use true virtual machines as containers! Thankfully to the open standards like OCI Runtime Spec, OCI Image Spec, or Kubernetes CRI, VM-based containers can be used by higher-level tools like _containerd_ and Kubernetes without requiring significant adjustments.

For more, check out this article on [how the OCI Runtime Spec defines a _standard container_](https://iximiuz.com/en/posts/oci-containers/):

![[Pasted image 20250113104126.png]]


# Conclusion 
Learning containers by only using high-level tools like Docker or Kubernetes can be both frustrating and dangerous. The domain is complex, and approaching it only from one end leaves too many grey areas.

The approach I find helpful starts from [taking a broader look at the ecosystem](https://iximiuz.com/en/posts/journey-from-containerization-to-orchestration-and-beyond/), decomposing it on layers, and then tackling them one by one starting from the bottom, and leveraging the knowledge obtained on every previous step:
- Container runtimes - Linux namespaces and cgroups.
- Container Images - why and how.
- Container Managers - making containers coexist on a single host.
- Container Orchestrators - combining multiple hosts into a single cluster.
- Container Standards - generalize the containers' knowledge.

