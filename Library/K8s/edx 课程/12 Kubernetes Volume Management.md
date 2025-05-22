在当今的商业模式中，数据是许多初创企业和企业最宝贵的资产。在 Kubernetes 集群中，Pod 中的容器可以是数据生产者、数据消费者，或者两者兼而有之。虽然某些容器数据预计是瞬时性的，并且预计不会超过 Pod 的寿命，但其他形式的数据必须超过 Pod 的寿命才能被聚合并加载到分析引擎中。Kubernetes 必须提供存储资源，以便提供容器消费的数据或存储容器生成的数据。Kubernetes 使用多种类型的卷和其他一些形式的存储资源来管理容器数据。在本章中，我们将讨论临时卷定义、PersistentVolume 和 PersistentVolumeClaim 对象，它们帮助我们将持久存储卷附加到 Pod。

By the end of this chapter, you should be able to:

- Explain the need for persistent data management.
- Compare Kubernetes Volume types.
- Discuss ephemeral Volumes, PersistentVolumes, and PersistentVolumeClaims.

# Volumes
众所周知，Pod 中运行的容器本质上是临时的。如果容器崩溃，容器内存储的所有数据都会被删除。但是，kubelet 会以干净的状态重新启动容器，这意味着容器内不会有任何旧数据。

为了解决这个问题，Kubernetes 使用了临时卷 (Ephemeral Volumes)，这是一种存储抽象，允许 Kubernetes 使用各种存储技术，并将其作为存储介质提供给 Pod 中的容器。临时卷本质上是容器文件系统上的一个挂载点，由存储介质支持。存储介质、内容和访问模式由卷类型决定。

In Kubernetes, an ephemeral Volume is linked to a Pod and can be shared among the containers of that Pod. Although the ephemeral Volume has the same life span as the Pod, meaning that it is deleted together with the Pod, the ephemeral Volume outlives the containers of the Pod - this allows data to be preserved across container restarts.


# Container Storage Interface (CSI)
Kubernetes、Mesos、Docker 或 Cloud Foundry 等容器编排器过去都有各自的方法，使用卷 (Volume) 来管理外部存储。对于存储供应商而言，为不同的编排器管理不同的卷插件是一项挑战。对于 Kubernetes 来说，这同样是一个可维护性挑战，因为它涉及集成到编排器源代码中的树内存储插件。来自不同编排器的存储供应商和社区成员开始合作，共同制定卷接口的标准化——这是一个使用标准化容器存储接口 (CSI) 构建的卷插件，旨在与各种存储提供商兼容的不同容器编排器。探索 CSI 规范了解更多详情。

在 Kubernetes v1.9 和 v1.13 版本之间，CSI 从 alpha 版本发展到稳定支持，这使得安装新的符合 CSI 标准的卷驱动程序变得非常容易。CSI 允许第三方存储提供商开发解决方案，而无需将其添加到核心 Kubernetes 代码库中。这些解决方案是 CSI 驱动程序，仅在集群管理员需要时安装。

# Volume Types
挂载在 Pod 内的目录由底层卷类型支持。卷类型决定了目录的属性，例如大小、内容、默认访问模式等。支持临时卷的一些卷类型示例如下：

- **emptyDir**  
    An **empty** Volume is created for the Pod as soon as it is scheduled on the worker node. The Volume's life is tightly coupled with the Pod. If the Pod is terminated, the content of **emptyDir** is deleted forever.  
- **hostPath**  
    The **hostPath** Volume Type shares a directory between the host and the Pod. If the Pod is terminated, the content of the Volume is still available on the host.
- **gcePersistentDisk**  
    The **gcePersistentDisk** Volume Type mounts a [Google Compute Engine (GCE) persistent disk](https://cloud.google.com/compute/docs/disks/) into a Pod.
- **awsElasticBlockStore**  
    The **awsElasticBlockStore** Volume Type mounts an [AWS EBS Volume](https://aws.amazon.com/ebs/) into a Pod. 
- **azureDisk**  
    The **azureDisk** mounts a [Microsoft Azure Data Disk](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/managed-disks-overview) into a Pod.
- **azureFile**  
    The **azureFile** mounts a [Microsoft Azure File Volume](https://github.com/kubernetes/examples/tree/master/staging/volumes/azure_file) into a Pod.
- **cephfs**  
    The **cephfs** allows for an existing [CephFS](https://ceph.io/ceph-storage/) volume to be mounted into a Pod. When a Pod terminates, the volume is unmounted and the contents of the volume are preserved.
- **nfs**  
    The **nfs** mounts an [NFS](https://github.com/kubernetes/examples/tree/master/staging/volumes/nfs) share into a Pod.
- **iscsi**  
    The **iscsi** mouns an [iSCSI](https://github.com/kubernetes/examples/tree/master/volumes/iscsi) share into a Pod.
- **secret**  
    The **[secret](https://kubernetes.io/docs/concepts/configuration/secret/)** Volume Type facilitates the supply of sensitive information, such as passwords, certificates, keys, or tokens to Pods.
- **configMap**  
    The **[configMap](https://kubernetes.io/docs/concepts/configuration/configmap/)** objects facilitate the supply of configuration data, or shell commands and arguments into a Pod.
- **persistentVolumeClaim**  
    A **[PersistentVolume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)** is consumed by a Pod using a **[persistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)**.

## PersistentVolumes
在典型的 IT 环境中，存储由存储/系统管理员管理。最终用户只会收到使用存储的指令，而无需参与底层存储管理。

在容器化世界中，我们希望遵循类似的规则，但考虑到我们之前看到的众多卷类型，这变得具有挑战性。Kubernetes 通过持久卷 (PV) 子系统解决了这个问题，该子系统为用户和管理员提供了用于管理和使用持久存储的 API。To manage the Volume, it uses the PersistentVolume API resource type, and to consume it, it uses the PersistentVolumeClaim API resource type.

持久卷是一种由多种存储技术支持的存储抽象，它可以位于部署 Pod 及其应用程序容器的主机本地、网络附加存储、云存储或分布式存储解决方案。持久卷由集群管理员静态配置。

![[Pasted image 20250515154548.png]]
**PersistentVolume**

持久卷 (PersistentVolume) 可以根据 StorageClass 资源进行动态配置。StorageClass 包含用于创建持久卷的预定义配置程序和参数。用户使用 PersistentVolumeClaims 发送动态 PV 创建请求，该请求会连接到 StorageClass 资源。

Some of the Volume Types that support managing storage using PersistentVolumes are:

- GCEPersistentDisk
- AWSElasticBlockStore
- AzureFile
- AzureDisk
- CephFS
- NFS
- iSCSI

## PersistentVolumeClaims
持久卷声明 (PVC) 是用户的存储请求。用户根据存储类别、访问模式、大小以及可选的卷模式请求持久卷资源。访问模式有四种：ReadWriteOnce（单节点读写）、ReadOnlyMany（多节点只读）、ReadWriteMany（多节点读写）和 ReadWriteOncePod（单 Pod 读写）。可选的卷模式（文件系统或块设备）允许将卷挂载到 Pod 的目录中或作为原始块设备。Kubernetes 的设计不支持对象存储，但可以借助自定义资源类型来实现。一旦找到合适的持久卷，就会将其绑定到持久卷声明。
![[Pasted image 20250515154836.png]]

**PersistentVolumeClaim**

After a successful bound, the PersistentVolumeClaim resource can be used by the containers of the Pod.
![[Pasted image 20250515154917.png]]
**PersistentVolumeClaim Used In a Pod**

Once a user finishes its work, the attached PersistentVolumes can be released. The underlying PersistentVolumes can then be _reclaimed_ (for an admin to verify and/or aggregate data), _deleted_ (both data and volume are deleted), or _recycled_ for future usage (only data is deleted), based on the configured **persistentVolumeReclaimPolicy** property.