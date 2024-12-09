Two key operating system abstractions: the process, which is a virtualization of the CPU, and the address space, which is a virtualization of memory. In tandem, these two abstractions allow a program to run as if it is in its own private, isolated; as if it has its own processor; as if it has its own memory.

# 39.1 Files and directories
Two key abstractions have developed over time in the virtualization of storage. The first is the **file**. The lower-level name of a file is often referred to as its **inode number** (**i-number**).

The second abstraction is that of a **directory**. 

The directory hierarchy starts at a **root directory** (in UNIX-based systems, the root directory is simply referred to as /) 

# 39.2 The file System Interface
We'll discover the mysterious call that is used to remove files, known as `unlink()`.

# 39.3 Creating Files
This can be accomplished with the `open` system call; by calling `open()` and passing it the `O_CREAT` flag, a program can create a new file. Here is some example code to create a file called "foo" in the current working directory:
```
int fd = open("foo", O_CREAT|O_WRONLY|O_TRUNC, S_IRUSR|S_IWUSR);
```

The routine `open()` takes a number of different flags. In this example, the second parameter creates the file (O_CREAT) if it does not exist, ensures that the file can only be written to (O_WRONLY), and, if the file already exists, truncates it to a size of zero bytes thus removing any existing content (O_TRUNC). The third parameter specifies permissions, in this case making the file readable and writable by the owner.

`open()` returns: a **file descriptor**. A file descriptor is just an integer, private per process, and is used in UNIX systems to access files; In this way, a file descriptor is a capability: an opaque handle that gives you the power to perform certain operations.

As stated above, file descriptors are managed by the operating system on a per-process basis. This means some kind of simple structure (e.g., an array) is kept in the `proc` structure on UNIX systems. Here is the relevant piece from the xv6 kernel:
```
struct proc{
	struct file *ofile[NOFILE]; // Open files
};
```
A simple array (with a maximum of NOFILE open files), indexed by the file descriptor, tracks which files are opened on a per-process basis. Each entry of the array is actually just a pointer to a struct file, which will be used to track information about the file being read or written.

## strace
The tool also takes some arguments which can be quite useful. For example, *-f* follows any fork'd children too; *-t* reports the time of day at each call; *-e trace=open,close,read,write* only traces calls to those system calls and ignores all others.

# 39.4 Reading And Writing Files
Here is an example of using *strace* to figure out what *cat* is doing:
![[Pasted image 20241202103745.png]]

The first thing that *cat* does is open the file for reading. A couple of things we should note about this; first, that the file is only opened for reading (not writing), as indicated by the *O_RDONLY* flag; second, that the 64-bit offset is used (O_LARGEFILE); third, that the call to *open()* succeeds and returns a file descriptor, which has the value of 3.

As it turns out, each running process already has three files open, standard input (which the process can read to receive input), standard output (which the process  can write to in order to dump information to the screen), and standard error (which the process can write error messages to). These are represented by file descriptors 0, 1, and 2, respectively. Thus, when you first open another file (as **cat** does above), it will almost certainly be file descriptor 3.

The second argument points to a buffer where the result of the **read()** will be placed; in the system-call trace above, strace shows the results of the read in this spot ("hello"). The third argument is the size of the buffer, which in this case is 4KB. The call to **read()** returns successfully as well, here returning the number of bytes it read (6, which includes 5 for the letters in the word "hello" and one for an end-of-line marker).

At this point, you see another interesting result of the strace: a single call to the `write()` system call.

The cat program then tries to read more from the file, but since there are no bytes left in the file, the `read()` returns 0 and the program knows that this means it has read the entire file. Thus, the program calls `close()` to indicate that it is done with the file "foo", passing in the corresponding file descriptor. The file is thus closed, and the reading of it thus complete.

# 39.5 Reading And Writing, But Not Sequentially
Thus far, we've discussed how to read and write files, but all access has been **sequential**; that is, we have either read a file from the beginning to the end, or written a file out from beginning to end.

Sometimes, however, it is useful to be able to read or write to a specific offset within a file; for example, if you build an index over a text document, and use it to look up a specific word, you may end up reading from some random offsets within the document. To do so, we will use the `lseek()` system call. Here is the function prototype:
```
off_t lseek(int fildes, off_t offset, int whence);
```
The second argument is the offset, which positions the file offset to a particular location within the file. The third argument, called whence for historical reasons, determines exactly how the seek is performed. From the man page:
- If whence is SEEK_SET, the offset is set to offset bytes.
- If whence is SEEK_CUR, the offset is set to its current location plus offset bytes.
- If whence is SEEK_END, the offset is set to the size of the file plus offset bytes.

Thus, part of the abstraction of an open file is that it has a current offset, which is updated in one of two ways. The first is when a read or write of N bytes takes place, N is added to the current offset; thus each read or write implicitly updates the offset. The second is explicitly with lseek, which changes the offset as specified above.

The offset, as you might have guessed, is kept in that *struct file* we saw earlier, as referenced from the *struct proc*.
```
struct file{
	int ref;
	char readable;
	char writable;
	struct inode *ip;
	uint off;
};
```

As you can see in the structure, the OS can use this to determine whether the opened file is readable or writable (or both), which underlying file it refers to (as pointed to by the `struct inode` pointer ip), and the current offset (off).

open file table:
```
struct{
	struct spinlock lock;
	struct file file[NFILE];
}ftable;
```

First, let's track a process that opens a file (of size 300 bytes) and reads it by calling the `read()` system call repeatedly, each time reading 100 bytes. Here is a trace of the relevant system calls, along with the values returned by each system call, and the value of the current offset in the open file table for this file access:
![[Pasted image 20241202111620.png]]

Second, let's trace a process that opens the `same` file twice and issues a read to each of them.
![[Pasted image 20241202111754.png]]

Final example, a process uses `lseek()` to reposition the current offset before reading; in this case, only a single open file table entry is needed.
![[Pasted image 20241202112017.png]]


## LSEEK()
The `lseek()` call simply changes a variable in OS memory that tracks, for a particular process, at which offset its next read of write will start. A disk seek occurs when a read or write issued to the disk is not on the same track as the last read or write, and thus necessitates a head movement.

# 39.6 Shared File Table Entries: fork() And dup()
在许多情况下（如上例所示），文件描述符与打开文件表中条目的映射是一对一映射。例如，当一个进程运行时，它可能会决定打开一个文件、读取它，然后关闭它；在这个例子中，该文件将在打开文件表中有一个唯一的条目。即使其他进程同时读取同一个文件，每个进程在打开文件表中都会有自己的条目。这样，文件的每次逻辑读取或写入都是独立的，并且在访问给定文件时，每个进程都有自己的当前偏移量。

However, there are a few interesting cases where an entry in the open file table is shared. One of those cases occurs when a parent process creates a child process with fork(). 
![[Pasted image 20241202140948.png]]
![[Pasted image 20241202141039.png]]

Figure 39.3 shows the relationships that connect each process's private descriptor array, the shared open file table entry, and the reference from it to the underlying file-system inode. Note that we finally make use of the **reference count** here. When a file table entry is shared, its reference count is incremented; only when both processes close the file (or exit) will the entry be removed.

![[Pasted image 20241202141221.png]]

One other interesting, and perhaps more useful, case of sharing occurs with the `dup()` system call (and its cousins, **dup2()** and **dup3()**).

dup() 调用允许进程创建一个新的文件描述符，该描述符引用与现有描述符相同的底层打开文件。图 39.4 显示了一小段代码，展示了如何使用 dup()。

dup() 调用（尤其是 dup2()）在编写 UNIX shell 并执行输出重定向等操作时很有用；花点时间想想为什么！现在，你在想：为什么我在做 shell 项目时他们没有告诉我这个？哦，好吧，即使在一本关于操作系统的令人难以置信的书中，你也无法让一切都按正确的顺序排列。对不起！

![[Pasted image 20241202141544.png]]

# 39.7 Writing Immediately With fsync()
大多数情况下，当程序调用 write() 时，它只是告诉文件系统：请在未来的某个时间点将此数据写入持久存储。出于性能原因，文件系统会在内存中缓冲此类写入一段时间（例如 5 秒或 30 秒）；在稍后的时间点，写入将实际发送到存储设备。从调用应用程序的角度来看，写入似乎很快完成，并且只有在极少数情况下（例如，机器在 write() 调用之后但在写入磁盘之前崩溃）才会丢失数据。

但是，有些应用程序需要的不仅仅是这种最终保证。例如，在数据库管理系统 (DBMS) 中，开发正确的恢复协议需要能够不时强制写入磁盘。

To support these types of applications, most file systems provide some additional control APIs. In the UNIX world, the interface provided to applications is known as `fsync(int fd)`. When a process calls `fsync()` for a particular file descriptor, the file system responds by forcing all dirty data to disk, for the file referred to by the specified file descriptor. The `fsync()` routine returns once all of these writes are complete.

The code opens the file `foo`, writes a single chunk of data to it, and then calls `fsync()` to ensure the writes are forced immediately to disk. Once the `fsync()` returns, the application can safely move on, knowing that the data has been persisted.
![[Pasted image 20241202142509.png]]


## Aside: mmap() and persistent memory
**Memory mapping** is an alternative way to access persistent data in files. The **mmap()** system call creates a correspondence between **byte offsets in a file** and **virtual addresses** in the calling process; the former is called the **backing file** and the latter its **in-memory image**. The process can then access the backing file using CPU instructions (i.e., loads and stores) to the in-memory image.

通过将文件的持久性与内存的访问语义相结合，文件支持的内存映射支持称为**持久内存**的软件抽象。持久内存风格的编程可以通过消除内存和存储的不同数据格式之间的转换来简化应用程序。

![[Pasted image 20241202143059.png]]

程序 pstack.c（包含在 OSTEP 代码 github repo 中，上面显示了一个片段）将持久堆栈存储在文件 ps.img 中，该文件最初是一袋零，例如，通过 truncate 或 dd 实用程序在命令行上创建。该文件包含堆栈大小的计数和保存堆栈内容的整数数组。在 mmap() 备份文件之后，我们可以使用指向内存映像的 C 指针访问堆栈，例如，p->n 访问堆栈上的项目数，p->stack 访问整数数组。由于堆栈是持久的，因此一次 pstack 调用推送的数据可以在下一次调用中弹出。

例如，在增量和推送分配之间发生的崩溃可能会导致我们的持久堆栈处于不一致的状态。应用程序通过使用针对故障以原子方式更新持久内存的机制来防止此类损坏。
# 39.8 Renaming Files
一旦我们有了一个文件，有时给文件起一个不同的名称是很有用的。在命令行中输入时，可以使用 mv 命令完成此操作；在此示例中，文件 foo 被重命名为 bar：
![[Pasted image 20241202142628.png]]
Using `strace`, we can see that `mv` uses the system call `rename(char *old, char *new)`.

rename() 调用提供的一个有趣的保证是，它（通常）作为与系统崩溃有关的原子调用实现；如果系统在重命名期间崩溃，则文件将被命名为旧名称或新名称，并且不会出现奇怪的中间状态。因此，rename() 对于支持某些需要对文件状态进行原子更新的应用程序至关重要。

# 39.9 Getting Information About Files
除了文件访问之外，我们还希望文件系统能够保存有关其存储的每个文件的大量信息。我们通常将有关文件的此类数据称为元数据。要查看某个文件的元数据，我们可以使用 stat() 或 fstat() 系统调用。这些调用将路径名（或文件描述符）传递给文件并填充 stat 结构，如图 39.5 所示。
![[Pasted image 20241202185534.png]]
![[Pasted image 20241202185649.png]]
每个文件系统通常将此类信息保存在名为 inode 的结构中。当我们讨论文件系统实现时，我们将学习更多关于 inode 的知识。现在，您应该将 inode 视为文件系统保存的持久数据结构，其中包含我们上面看到的信息。所有 inode 都驻留在磁盘上；活动 inode 的副本通常会缓存在内存中以加快访问速度。

# 39.10 Removing Files
![[Pasted image 20241202185821.png]]

 我们从跟踪的输出中删除了一堆不相关的垃圾，只留下一个神秘的系统调用 unlink()。如您所见，unlink() 只接受要删除的文件的名称，并在成功时返回零。但这给我们带来了一个大难题：为什么
这个系统调用名为 unlink？为什么不直接删除或删除？要理解这个难题的答案，我们必须首先了解文件以外的内容，还要了解目录。

# 39.11 Making Directories
除了文件之外，一组与目录相关的系统调用使您能够创建、读取和删除目录。请注意，您永远无法直接写入目录。由于目录的格式被视为文件系统元数据，因此文件系统认为自己对目录数据的完整性负责；因此，您只能通过例如在目录中创建文件、目录或其他对象类型来间接更新目录。

To create a directory, a single system call, `mkdir()`, is available. The eponymous `mkdir` program can be used to create such a directory.
![[Pasted image 20241202190122.png]]

当创建这样的目录时，它被视为“空”，尽管它确实包含最少的内容。具体来说，空目录有两个条目：一个条目引用自身，一个条目引用其父目录。前者称为“.”（点）目录，后者称为“.”（点-点）。您可以通过将标志 (-a) 传递给程序 ls 来查看这些目录.

# 39.12 Reading Directories
`ls`. Let's write our own little tool like `ls` and see how it done. The program uses three calls, `opendir()`, `readdir()`, and `closedir()`, to get the job done.
![[Pasted image 20241202190719.png]]
The declaration below shows the information available within each directory entry in the `struct dirent` data structure:
![[Pasted image 20241202190841.png]]
由于目录信息量较少（基本上只是将名称映射到 inode 编号，以及一些其他详细信息），因此程序可能希望对每个文件调用 stat() 以获取有关每个文件的更多信息，例如其长度或其他详细信息。事实上，这正是当您将 -l 标志传递给 ls 时所做的；尝试使用 strace 来查看是否带有该标志。

# 39.13 Deleting Directories
`rmdir()` to delete a directory. Unlike file deletion, however, removing directories is more dangerous, as you could potentially delete a large amount of data with a single command. Thus, `rmdir()` has the requirement that the directory be empty before it is deleted. If you try to delete a non-empty directory, the call to `rmdir()` simply will fail.

# 39.14 Hard Links
现在，我们回到为什么通过 unlink() 删除文件的谜题，通过了解一种在文件系统树中创建条目的新方法，即通过称为 link() 的系统调用。link() 系统调用接受两个参数，一个旧路径名和一个新路径名；当您将新文件名“链接”到旧文件名时，您实际上是创建了另一种引用同一文件的方法。命令行程序 `ln` 用于执行此操作，如我们在此示例中所见：
![[Pasted image 20241202191433.png]]

Here we created a file with the word "hello" in it, and called the file `file`. We then create a hard link to that file using the `ln` program. After this, we can examine the file by either opening `file` or `file2`.

The way `link()` works is that it simply creates another name in the directory you are creating the link to, and refers it to the same inode number of the original file. 
![[Pasted image 20241202191842.png]]

Just make a new reference to the same exact inode number (67158084 in this example).

When you create a file, you are really doing two things. First, you are making a structure (the inode) that will track virtually all relevant information about the file, including its size, where its blocks are on disk, and so forth. Second, you are linking a human-readable name to that file, and putting that link into a directory.

在创建文件的硬链接后，文件系统感觉不到原始文件名（文件）和新创建的文件名（文件2）之间的区别；实际上，它们都只是指向文件底层元数据的链接，该元数据位于 inode 编号 67158084 中。

Thus, to remove a file from the file system, we call `unlink()`. In the example above, we could for example remove the file named `file`, and still access the file without difficulty:
![[Pasted image 20241202192429.png]]
之所以有效，是因为当文件系统取消文件链接时，它会检查 inode 编号内的引用计数。此引用计数（有时称为链接计数）允许文件系统跟踪有多少不同的文件名已链接到此特定 inode。当调用 unlink() 时，它会删除人类可读名称（正在删除的文件）与给定 inode 编号之间的“链接”，并减少引用计数；只有当引用计数达到零时，文件系统才会释放 inode 和相关数据块，从而真正“删除”文件。

You can see the reference count of a file using `stat()` of course. Let’s see what it is when we create and delete hard links to a file. In this example, we’ll create three links to the same file, and then delete them. Watch the link count!

![[Pasted image 20241202192634.png]]

# 39.15 Symbolic Links
还有一种非常有用的链接，称为符号链接，有时也称软链接。硬链接有一定限制：
您无法创建指向目录的链接（因为担心会在目录树中创建循环）；您无法硬链接到其他磁盘分区中的文件（因为 inode 编号仅在特定文件系统内是唯一的，而不是跨文件系统）；等等。因此，创建了一种称为符号链接的新链接类型。

To create such a link, you can use the same program `ln`, but with the `-s` flag. Here is an example:
![[Pasted image 20241202192937.png]]
第一个区别是符号链接本身其实就是一个文件，属于不同的类型。我们已经讨论过常规文件和目录；符号链接是文件系统知道的第三种类型。符号链接的统计数据揭示了一切：
![[Pasted image 20241202193139.png]]

运行 ls 也揭示了这一事实。如果仔细查看 ls 输出的长格式的第一个字符，您会发现最左列中的第一个字符是 -（表示常规文件）、d（表示目录）和 l（表示软链接）。您还可以看到符号链接的大小（本例中为 4 个字节）以及链接指向的内容（名为 file 的文件）。
![[Pasted image 20241202193221.png]]

file2 为 4 个字节的原因在于，符号链接的形成方式是将链接文件的路径名作为链接文件的数据。由于我们链接到了一个名为 `file` 的文件，因此我们的链接文件 `file2` 很小（4 个字节）。如果我们链接到更长的路径名，我们的链接文件会更大：
![[Pasted image 20241202193338.png]]
Finally, because of the way symbolic links are created, they leave the possibility for what is known as a **dangling reference**:
![[Pasted image 20241202193448.png]]

# 39.16 Permission Bits And Access Control Lists
进程的抽象提供了两个核心虚拟化：CPU 虚拟化和内存虚拟化。每个虚拟化都给进程一种错觉，认为它拥有自己的私有 CPU 和私有内存；实际上，底层操作系统使用各种技术以安全可靠的方式在竞争实体之间共享有限的物理资源。

文件系统还提供了磁盘的虚拟视图，将其从一堆原始块转换为更方便用户使用的文件和目录，如本章所述。但是，这种抽象与 CPU 和内存的抽象明显不同，因为文件通常在不同的用户和进程之间共享，并且并非（总是）私有的。因此，文件系统中通常存在一套更全面的机制来实现不同程度的共享。

The first form of such mechanisms is the classic UNIX **permission bits**. To see permissions for a file `foo.txt`, just type:
![[Pasted image 20241202193817.png]]

第一个 `-` 代表是一个普通文件，`d` 代表目录，`l` 代表是符号链接。

We are interested in the permission bits, which are represented by the next nine characters (rw-r--r--). The permissions consist of three groupings: what the `owner` of the file can do to it, what someone in a `group` can do to the file, and finally, what anyone (sometimes referred to as `other`) can do. Read, write, execute.

In the example above, the first three characters of the output of `ls` show that the file is both readable and writable by the owner (rw-), and only readable by members of the group `wheel` and also by anyone else in the system (r--).

The owner of the file can readily change these permissions, for example by using the `chmod` command (to change the `file mode`). To remove the ability for anyone except the owner to access the file, you could type:
![[Pasted image 20241202194449.png]]


除了权限位之外，某些文件系统（例如称为 AFS 的分布式文件系统（将在后面的章节中讨论））还包含更复杂的控制。例如，AFS 以每个目录的访问控制列表 (ACL) 的形式实现这一点。访问控制列表是一种更通用且更强大的方式，可以准确表示谁可以访问给定的资源。在文件系统中，这使用户能够创建一个非常具体的列表，说明谁可以和不能读取一组文件，这与上面描述的权限位的有些有限的所有者/组/每个人模型形成鲜明对比。

# 39.17 Making and Mounting A File System
How to assemble a full directory tree from many underlying file systems. This task is accomplished via first making file systems, and then mounting them to make their contents accessible.

To make a file system, most file systems provide a tool, usually referred to as `mkfs`, that performs exactly this task. The idea is as follows: give the tool, as input, a device (such as a disk partition, e.g., /dev/sda1) and a file system type (e.g., ext3), and it simply writes an empty file system, starting with a root directory, onto that disk partition. And mkfs said, let there be a file system!

However, once such a file system is created, it needs to be made accessible within the uniform file-system tree. This task is acieved via the `mount` program (which makes the underlying system call `mount()` to do the real work). What mount does, quite simply is take an existing directory as a target `mount point` and essentially paste a new file system onto the directory tree at that point.

An example here might be useful. Imagine we have an unmounted ext3 file system, stored in device partition `/dev/sda1`, that has the following contents: a root directory which contains two sub-directories, `a` and `b`, each of which in turn holds a single file named `foo`. Let's say we wish to mount this file system at the mount point `/home/users`. We would type something like this:
![[Pasted image 20241202200518.png]]
If successful, the mount would thus make this new file system available. However, note how the new file system is now accessed. To look at the contents of the root directory, we would use `ls` like this:
![[Pasted image 20241202200635.png]]

As you can see, the pathname `/home/users/` now refers to the root of the newly-mounted directory. Similarly, we could access directories `a` and `b` with the pathnames `/home/users/a` and `/home/users/b`. And thus the beauty of mount: instead of having a number of separate file systems, mount unifies all file systems into one tree, making naming uniform and convenient.

To see what is mounted on your system, and at which points, simply run the `mount` program. You'll see:
![[Pasted image 20241202201217.png]]
这种疯狂的混合表明，大量不同的文件系统，包括 ext3（一种基于磁盘的标准文件系统）、proc 文件系统（一种用于访问有关当前进程的信息的文件系统）、tmpfs（一种仅用于临时文件的文件系统）和 AFS（一种分布式文件系统）都粘合在这台机器的文件系统树上。

# 39.18 Summary
- Each file descriptor is a private, per-process entity, which refers to an entry in the `open file table`. The entry therein tracks which file this access refers to, the `current offset` of the file and other.
- To force updates to persistent media, a process must use `fsync()` or related calls. However, doing so correctly while maintaining high performance is challenging, so think carefully when doing so.
- And remember, deleting a file is just performing that one last `unlink()` of it from the directory hierarchy.
- Most file systems have mechanisms to enable and disable sharing. A rudimentary form of such controls are provided by `permissions bits`; more sophisticated `access control lists` allow for more precise control over exactly who can access and manipulate information.

