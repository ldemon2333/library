# Brief container history: from a mess to standards and proper architecture
Container technologies appeared for the first time after the invention of _cgroups_ and _namespaces_. Two of the first-known projects trying to combine them to achieve isolated process environments were [LXC](https://linuxcontainers.org/lxc/introduction/) and [LMCTFY](https://github.com/google/lmctfy). The latter, in a typical _Google_ style, tried to provide a stable abstraction from the Linux internals via an intuitive API.

In 2013 a technology named _Docker_, built on top of _LXC_, was created. The _Docker_ team introduced the notions of container “packaging” (called images) and “portability” of these images between different machines. In other words, _Docker_ tried to create an independent software component following the Unix philosophy (minimalism, modularity, interoperability).

In 2014 a software called [libcontainer](https://github.com/docker-archive/libcontainer) was created. Its objectives were to create processes into isolated environments and manage their lifecycle. Also in 2014, [Kubernetes](https://kubernetes.io/) was announced at [DockerCon](https://dockerconeurope2014.sched.com/list/descriptions/) and that is when a lot of things started to happen in the container world.

The [OCI (Open Container Initiative)](https://opencontainers.org/) was then created in response to the need for standardization and structured governance. The _OCI_ project ended up with two specifications - the Runtime Specification (_runtime-spec_) and the Image Specification (_image-spec_). The former defined a detailed API for the developers of **runtimes** to follow. The _libcontainer_ project was donated to _OCI_ and the first standardized runtime following the _runtime-spec_ was created - **runc**. It represents a fully compatible API on top of libcontainer allowing users to directly spawn and manage containers.

Today container runtimes are often divided into two categories - low-level (runc, gVisor, Firecracker) and high-level (containerd, Docker, CRI-O, podman). The difference is in the amount of consumed _OCI_ specification and additional features.

_Runc_ is a standardized runtime for spawning and running containers on Linux according to the _OCI_ specification. However, it doesn’t follow the _image-spec_ specification of the _OCI_.

There are other more high-level runtimes, like _Docker_ and _Containerd_, which implement this specification on top of _runc_. By doing so, they solve several disadvantages related to the usage of _runc_ alone namely - image integrity, bundle history log and efficient container root file system usage.

![[Pasted image 20250207221715.png]]

# Runc
As mentioned above, _runc_ is an _OCI_ compliant runtime - a software component responsible for the creation, configuration and management of isolated Linux processes also called containers. Formally, _runc_ is a client wrapper around _libcontainer_. As _runc_ follows the _OCI_ specification for container runtimes it requires two pieces of information:
- **OCI configuration** - a JSON-like file containing container process information like namespaces, capabilities, environment variables, etc.
- **Root filesystem directory** - the root file system directory which is going to be used by the container process (chroot).

## Generating an OCI configuration
_Runc_ comes out of the box with a feature to generate a default _OCI_ configuration:
```
runc spec
```
From the above code snippet, one can see that the _config.json_ file contains information about:
- [capabilities](https://man7.org/linux/man-pages/man7/capabilities.7.html);
    
- [task structure info](https://docs.huihoo.com/doxygen/linux/kernel/3.7/structtask__struct.html);
    
- [mount points](https://www.linuxtopia.org/online_books/introduction_to_linux/linux_Mount_points.html);
    
- [namespaces](https://man7.org/linux/man-pages/man7/namespaces.7.html);
    
- [cgroups](https://man7.org/linux/man-pages/man7/cgroups.7.html).

All of the above configuration is almost completely sufficient for _libcontainer_ to create an isolated _Linux_ process (container). There is one thing missing to spawn a container - the process root directory. Let’s download one and make it available for _runc_.


We will distinguish two types of containers: - **root** - containers running under UID=0. - **rootless** - containers running under a UID different from 0.

## Creating root container
Our baby container is now created but it is not running the defined _sh_ program as configured in the _JSON_ file. That is because the _runc init_ process is still keeping the process which has to load the _sh_ program in a namespaced (isolated) environment. Let’s inspect what is going on inside _runc init_.

To understand why your container is not running the defined `sh` program as expected, we need to dive into how `runc` works and how it initializes a container. The issue you're describing seems to be related to how `runc init` is handling the process and the namespaces that are involved.

# runc source architecture
Following shows the source code architecture of runc
![[Pasted image 20250207230147.png]]
The runc binary has several subcommands, every handler is in the go file of root directory. The core code of runc is the libcontainer directory. In the next post I will analysis the runc create and start command.

