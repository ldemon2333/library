创建文件系统和挂载文件系统
# file_system_type
```c
struct file_system_type {
        const char *name;
        int fs_flags;
#define FS_REQUIRES_DEV         1 
#define FS_BINARY_MOUNTDATA     2
#define FS_HAS_SUBTYPE          4
#define FS_USERNS_MOUNT         8       /* Can be mounted by userns root */
#define FS_RENAME_DOES_D_MOVE   32768   /* FS will handle d_move() during rename() internally. */
        struct dentry *(*mount) (struct file_system_type *, int,
                       const char *, void *);
        void (*kill_sb) (struct super_block *);
        struct module *owner;
        struct file_system_type * next;
        struct hlist_head fs_supers;

        struct lock_class_key s_lock_key;
        struct lock_class_key s_umount_key;
        struct lock_class_key s_vfs_rename_key;
        struct lock_class_key s_writers_key[SB_FREEZE_LEVELS];

        struct lock_class_key i_lock_key;
        struct lock_class_key i_mutex_key;
        struct lock_class_key i_mutex_dir_key;
};
```

name: 文件系统的名字

fs_flags: 说明文件系统的类型，下面的宏定义代表了它的几种类型：
- FS_REQUIRES_DEV: 文件系统必须在物理设备上。
- FS_BINARY_MOUNTDATA: mount此文件系统时（参见mount_fs函数 - fs/super.c）需要使用二进制数据结构的mount data（如每个位域都有固定的位置和意义），常见的nfs使用这种mount data（参见struct nfs_mount_data结构 - include/uapi/linux/nfs_mount.h）。
- FS_HAS_SUBTYPE: 文件系统含有子类型，最常见的就是FUSE，FUSE本是不是真正的文件系统，所以要通过子文件系统类型来区别通过FUSE接口实现的不同文件系统。
- FS_USERNS_MOUNT: 文件系统每次挂载都后都是不同的user namespace，如用于devpts。
- FS_RENAME_DOES_D_MOVE: 文件系统将把重命名操作reame()直接按照移动操作d_move()来处理，主要用于网络文件系统。

mount: 代替早期的get_sb()，用户挂载此文件系统时使用的回调函数。

kill_sb: 删除内存中的super block，在卸载文件系统时使用。

owner: 指向实现这个文件系统的模块，通常为THIS_MODULE宏。

next: 指向文件系统类型链表的下一个文件系统类型。

fs_supers: 此文件系统类型的文件系统超级块结构都串连在这个表头下。

对于一个文件系统，定义好自己的file_system_type结构后就需要将自己注册进内核。

# register_filesystem
参考fs/filesystem.c，这个文件不长，在开头有一个重要的全局变量：

```c
static struct file_system_type *file_systems;
```

这个指针将指向所有已注册的文件系统模块。

看过这个全局变量后我们接着看register_filesystem函数，其定义与解释都也很详细，如下(来自Linux 4.17-rc2, fs/filesystem.c)

```c
/**
 *      register_filesystem - register a new filesystem
 *      @fs: the file system structure
 *
 *      Adds the file system passed to the list of file systems the kernel
 *      is aware of for mount and other syscalls. Returns 0 on success,
 *      or a negative errno code on an error.
 *
 *      The &struct file_system_type that is passed is linked into the kernel 
 *      structures and must not be freed until the file system has been
 *      unregistered.
 */
 
int register_filesystem(struct file_system_type * fs)
{
        int res = 0;
        struct file_system_type ** p;

        BUG_ON(strchr(fs->name, '.'));
        if (fs->next)
                return -EBUSY;
        write_lock(&file_systems_lock);
        p = find_filesystem(fs->name, strlen(fs->name));
        if (*p)
                res = -EBUSY;
        else
                *p = fs;
        write_unlock(&file_systems_lock);
        return res;
}
```

里面最主要的逻辑是find_filesystem调用及指针p的处理。先来看find_filesystem的定义：

```c
static struct file_system_type **find_filesystem(const char *name, unsigned len)
{
        struct file_system_type **p;
        for (p = &file_systems; *p; p = &(*p)->next)
                if (strncmp((*p)->name, name, len) == 0 &&
                    !(*p)->name[len])
                        break;
        return p;
}
```

在返回register_filesystem函数后，判断返回值，如果找到重复的则返回EBUSY错误，如果没找到重复的，就把当前要注册的文件系统挂到尾端file_system_type的next指针上，串联进链表。至此一个文件系统模块就注册好了。

当然与register相对应的就是unregister函数，这里就不说它了。在fs/filesystem.c里还有一个很重要的函数是get_fs_type()，它接受一个文件系统的名称作为参数，然后反向找到其对应的file_system_type实例，这个在mount的时候是很常用的。

说了怎么注册一个文件系统，那一个具体的文件系统的file_systemtype长什么样呢?

```c
static struct file_system_type xfs_fs_type = {
        .owner                  = THIS_MODULE,
        .name                   = "xfs",
        .mount                  = xfs_fs_mount,
        .kill_sb                = kill_block_super,
        .fs_flags               = FS_REQUIRES_DEV,
};
```

这是XFS的file_systemtype的实例，name表明其是xfs，fsflags表明其是必须用在存储设备上的文件系统，不是虚拟文件系统等。最重要的两个回调函数"mount"和"killsb"就是两个需要XFS模块来实现的带有具体细节的函数了。mount是在mount操作时会被调用的函数，kill_sb一般会在umount的时候使用。这里就不展开展开具体细节了，留到后面说到具体mount操作时再说。

至此我们了解了file_system__type，已经如何注册使用。一个file__system_type最主要告诉内核三件事：“我叫什么”，“怎么挂载我”以及“怎么卸载我”（多么纯粹的让人感动:）。


# mount
回忆一下，一个文件系统的file_system_type里有两个主要成员，一个是文件系统的名字，一个是mount这个文件系统的方法（其它参数也很重要，但重点就是这两个）。名字就是一个id，唯一标记一个文件系统，并方便内核在需要时根据它找到这个文件系统的file_system_type。mount方法在挂载这个文件系统时使用的，在需要挂载一个文件系统时通过name找到这个文件系统的file_system_type实例，然后使用这个实例中挂载的方法(mount函数)，进行挂载。

当一个文件系统被挂载之后，一个文件系统实例就诞生并可以使用了。

一般我们类似这样挂载一个文件系统：
```text
# mount -t xfs /dev/sdb1 /mnt -o ...
```
-t指定/dev/sdb1上的文件系统类型，如果不使用-t则mount命令也可以尝试探测device上的文件系统类型。

-o用来指定一些额外的（非默认的）挂载选项。

上面这个命令翻译成人话就是：请把/dev/sdb1 的 XFS 文件系统挂载到/mnt上，并在挂载时使能-o里的特性。

这是从一个命令的角度来看mount操作，那么一个命令的最终执行还得是陷入内核后执行的系统调用，来看一下mount系统调用的定义(man 2 mount):
```text
NAME
       mount - mount filesystem

SYNOPSIS
       #include <sys/mount.h>

       int mount(const char *source, const char *target,
                 const char *filesystemtype, unsigned long mountflags,
                 const void *data);
```

和上面的命令行对应一下，source对应/dev/sdb1，target对应/mnt，filesystemtype对应-t xfs，但是还剩下两个参数mountflags和data，可是我们只剩下一个-o选项了，这是怎么回事呢？

别急，我们先抛开mountflags和data看一下mount系统调用怎么用。

- 首先，我们在一个存储设备上创建一个文件系统

```text
# mkfs.xfs -f /dev/sdb1
```

- 然后我们准备一个挂载点

```text
# mkdir /mnt/scratch
```

- 最后我们使用mount系统调用来挂载上面的文件系统和挂载点

```text
# cat mymount.c
#include <sys/mount.h>  
#include <stdio.h>  
  
int main(int argc, char *argv[]) {  
        if (mount("/dev/sdb1", "/mnt/scratch", "xfs", 0, NULL)) {  
                perror("mount failed");  
        }
        return 0;  
}

# gcc -Wall -o mymount mymount.c
# ./mymount 
# cat /proc/mounts |grep sdb1
/dev/sdb1 /mnt/scratch xfs rw,seclabel,relatime,attr2,inode64,noquota 0 0
```

用mount系统调用的前三个参数，不带任何挂载选项就挂载了文件系统。好像蛮简单的，别着急，后面还两个可选参数呢。

