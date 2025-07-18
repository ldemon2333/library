development and ops teams 的矛盾，面对微服务这种架构，于是产生新技术——k8s

### Moving to continuous delivery: DevOps and NoOps
In the last few years, we’ve also seen a shift in the whole application development pro-
cess and how applications are taken care of in production. In the past, the develop-
ment team’s job was to create the application and hand it off to the operations team,
who then deployed it, tended to it, and kept it running. But now, organizations are
realizing it’s better to have the same team that develops the application also take part
in deploying it and taking care of it over its whole lifetime. This means the developer,
QA, and operations teams now need to collaborate throughout the whole process.
This practice is called DevOps.

### Letting Developers and Sysadmins do what they do best
Developers love creating new features and improving the user experience. They don’t normally want to be the ones making sure that the underlying operating system is up to date with all the security patches and things like that. They prefer to leave that up to the system administrators.

NoOps

K8s enables us to achieve all of this. 实现了 NoOps

![[Pasted image 20250712112849.png]]

Docker-based container images and VM images is that container images are composed of layers, which can be shared and reused across multiple images.



If a containerized application requires a specific kernel version, it may not work on every machine. If a machine runs a different version of the Linux kernel or doesn't have the same kernel modules available, the app can't run on it.

## Docker Image Layers: Efficiency in Storage and Distribution

Docker images are built in layers, which offers significant advantages for both **storage and network distribution**.

**Key benefits of layers:**

- **Efficient Distribution:** When you transfer Docker images, only the layers that are new or haven't been previously transferred need to be sent. If multiple images share the same base layers, those shared layers are only transferred once, speeding up the distribution process.
    
- **Reduced Storage Footprint:** Each layer is stored only once on a system, regardless of how many images or containers use it. This significantly reduces the disk space required for multiple images.
    
- **Container Isolation:** While containers can share the underlying read-only layers, they maintain isolation. When a container is launched, a new, writable layer is created on top of the read-only image layers. Any changes a container makes to a file in a lower layer result in a copy of that file being created in its top writable layer. This ensures that changes made by one container are not seen by others, even if they originated from the same base image.
    

---

## Portability Limitations of Container Images

While Docker containers are highly portable, they do have some limitations, primarily due to their reliance on the host system's resources:

**Key portability considerations:**

- **Host Linux Kernel Dependency:** All containers running on a host share and utilize the **host's Linux kernel**. This means if a containerized application requires a specific kernel version or particular kernel modules, it might not run successfully on a host with a different kernel version or missing modules. Virtual machines (VMs) overcome this limitation by running their own independent kernels.
    
- **Hardware Architecture Dependency:** A containerized application is built for a specific **hardware architecture** (e.g., x86, ARM). You cannot run an application containerized for one architecture (e.g., x86) on a machine with a different architecture (e.g., ARM), even if Docker is installed on both. Running applications across different hardware architectures typically still requires a virtual machine.
    


Borg and Omega