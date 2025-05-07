但是，默认情况下，Docker会为你提供一个隐含的ENTRYPOINT，即：/bin/sh -c。所以，在不指定ENTRYPOINT时，比如在我们这个例子里，实际上运行在容器里的完整进程是：`/bin/sh -c "python app.py"`，即CMD的内容就是ENTRYPOINT的参数。

更重要的是，由于执行这个挂载操作时，“容器进程”已经创建了，也就意味着此时Mount Namespace已经开启了。所以，这个挂载事件只在这个容器里可见。你在宿主机上，是看不见容器内部的这个挂载点的。这就**保证了容器的隔离性不会被Volume打破**。

> 注意：这里提到的”容器进程”，是Docker创建的一个容器初始化进程(dockerinit)，而不是应用进程(ENTRYPOINT + CMD)。dockerinit会负责完成根目录的准备、挂载设备和目录、配置hostname等一系列需要在容器内进行的初始化操作。最后，它通过execv()系统调用，让应用进程取代自己，成为容器里的PID=1的进程。

而这里要使用到的挂载技术，就是Linux的**绑定挂载（bind mount）机制**。它的主要作用就是，允许你将一个目录或者文件，而不是整个设备，挂载到一个指定的目录上。并且，这时你在该挂载点上进行的任何操作，只是发生在被挂载的目录或者文件上，而原挂载点的内容则会被隐藏起来且不受影响。

其实，如果你了解Linux 内核的话，就会明白，绑定挂载实际上是一个inode替换的过程。在Linux操作系统中，inode可以理解为存放文件内容的“对象”，而dentry，也叫目录项，就是访问这个inode所使用的“指针”。

![[Pasted image 20250506211304.png]]

正如上图所示，mount –bind /home /test，会将/home挂载到/test上。其实相当于将/test的dentry，重定向到了/home的inode。这样当我们修改/test目录时，实际修改的是/home目录的inode。这也就是为何，一旦执行umount命令，/test目录原先的内容就会恢复：因为修改真正发生在的，是/home目录里。

**所以，在一个正确的时机，进行一次绑定挂载，Docker就可以成功地将一个宿主机上的目录或文件，不动声色地挂载到容器中。**

这样，进程在容器里对这个/test目录进行的所有操作，都实际发生在宿主机的对应目录（比如，/home，或者/var/lib/docker/volumes/[VOLUME_ID]/\_data）里，而不会影响容器镜像的内容。

![[Pasted image 20250506212224.png]]
这个容器进程`“python app.py”`，运行在由Linux Namespace和Cgroups构成的隔离环境里；而它运行所需要的各种文件，比如python，`app.py`，以及整个操作系统文件，则由多个联合挂载在一起的rootfs层提供。

这些rootfs层的最下层，是来自Docker镜像的只读层。

在只读层之上，是Docker自己添加的Init层，用来存放被临时修改过的/etc/hosts等文件。

而rootfs的最上层是一个可读写层，它以Copy-on-Write的方式存放任何对只读层的修改，容器声明的Volume的挂载点，也出现在这一层。




