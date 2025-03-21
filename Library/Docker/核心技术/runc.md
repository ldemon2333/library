# history
Today container runtimes are often divided into two categories - low-level (runc, gVisor, Firecracker) and high-level (containerd, Docker, CRI-O, podman). The difference is in the amount of consumed _OCI_ specification and additional features.

_Runc_ is a standardized runtime for spawning and running containers on Linux according to the _OCI_ specification. However, it doesn’t follow the _image-spec_ specification of the _OCI_.

There are other more high-level runtimes, like _Docker_ and _Containerd_, which implement this specification on top of _runc_. By doing so, they solve several disadvantages related to the usage of _runc_ alone namely - image integrity, bundle history log and efficient container root file system usage. Be prepared, we are going to look into these more evaluated runtimes in the next article !

# Runc
Formally, _runc_ is a client wrapper around _libcontainer_. As _runc_ follows the _OCI_ specification for container runtimes it requires two pieces of information:
- **OCI configuration** - a JSON-like file containing container process information like namespaces, capabilities, environment variables, etc.
    
- **Root filesystem directory** - the root file system directory which is going to be used by the container process (chroot).

From the above code snippet, one can see that the _config.json_ file contains information about:

- [capabilities](https://man7.org/linux/man-pages/man7/capabilities.7.html);
    
- [task structure info](https://docs.huihoo.com/doxygen/linux/kernel/3.7/structtask__struct.html);
    
- [mount points](https://www.linuxtopia.org/online_books/introduction_to_linux/linux_Mount_points.html);
    
- [namespaces](https://man7.org/linux/man-pages/man7/namespaces.7.html);
    
- [cgroups](https://man7.org/linux/man-pages/man7/cgroups.7.html).

Okay, let’s put everything together in this bundle and spawn a new baby container! We will distinguish two types of containers: - **root** - containers running under UID=0. - **rootless** - containers running under a UID different from 0.

## Creating root container
Almost everything is ready. We have one last detail to take care of: since we would like to detach our root container from _runc_ and its file descriptors to interact with it, we have to create a _TTY_ socket and connect to it from both sides (from the container side and from our terminal). We are going to use [recvtty](https://github.com/opencontainers/runc/blob/main/contrib/cmd/recvtty/recvtty.go) which is part of the official **runc** project.
![[Pasted image 20250219225518.png]]
Now we switch to another terminal to create the container via _runc_.
![[Pasted image 20250219225618.png]]
Our baby container is now created but it is not running the defined _sh_ program as configured in the _JSON_ file. That is because the _runc init_ process is still keeping the process which has to load the _sh_ program in a namespaced (isolated) environment. Let’s inspect what is going on inside _runc init_.
![[Pasted image 20250219230103.png]]
We can confirm that, except for the _user_ and _time_ namespaces, the _init runc_ process is using a different set of namespaces than our regular shell. But why hasn’t it launched our shell process?

Here is the time to say again that _runc_ is a pretty low-level runtime and does not handle a lot of configurations compared to other more high-level runtimes. For example, it doesn’t configure any networking interface for the isolated process. In order to further configure the container process, _runc_ puts it in a containment facility (runc-init) where additional configurations can be added. To illustrate in a more practical way why this containment can be useful, we are going to refer to a previous [article](https://blog.quarkslab.com/digging-into-linux-namespaces-part-2.html) and configure the network namespace for our baby container.

## Configuring runc-init with network interface
![[Pasted image 20250219230658.png]]


**This exercise shows that if we want to keep control of the container's _stdio_ streams, the container process cannot be independent of the launching process.** And since we know, that the container manager can be restarted due to a crash, update, or some other reasons, that makes it impossible to launch _runc_ directly from the container manager process. Thus, we need to have a helper process living as long as the underlying container process and serving it. And such a process called **container runtime shim**.