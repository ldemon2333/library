在本章中，我们介绍一种简单的文件系统实现，称为 vsfs（非常简单的文件系统）。此文件系统是典型 UNIX 文件系统的简化版本，因此可用于介绍当今许多文件系统中都会出现的一些基本磁盘结构、访问方法和各种策略。

文件系统是纯软件；与我们的 CPU 和内存虚拟化开发不同，我们不会添加硬件功能来使文件系统的某些方面更好地工作（尽管我们希望关注设备特性以确保文件系统正常工作）。由于我们在构建文件系统方面具有很大的灵活性，因此已经构建了许多不同的文件系统，从 AFS（Andrew 文件系统） 到 ZFS（Sun 的 Zettabyte 文件系统）。所有这些文件系统都有不同的数据结构，并且在某些方面做得比同类更好或更差。因此，我们将通过案例研究来学习文件系统：首先，本章中的一个简单的文件系统（vsfs）来介绍大多数概念，然后对一系列真实的文件系统进行研究，以了解它们在实践中如何不同。

How to implement a simple file system? how can we build a simple file system? What structures are needed on the disk? What do they need to track? How are they accessed?

# 40.1 The way to think
在思考文件系统时，我们通常建议从两个不同的方面来思考；如果你理解了这两个方面，你就可能理解了文件系统的基本工作原理。

The first is the **data structures** of the file system. 换句话说，文件系统使用什么类型的磁盘结构来组织其数据和元数据? The first file systems we'll see employ simple structures, like arrays of blocks or other objects, use tree-based structures.

The second aspect of a file system is its **access methods**. 它如何将进程的调用（例如 open()、read()、write() 等）映射到其结构上？在执行特定系统调用期间，会读取哪些结构？会写入哪些结构？所有这些步骤的执行效率如何？

如果您了解文件系统的数据结构和访问方法，那么您就已建立了一个关于其真正工作原理的良好心理模型，这是系统思维的一个关键部分。在我们深入研究第一个实现时，请尝试开发您的心理模型。

# 40.2 Overall Organization
现在，我们来开发 vsfs 文件系统数据结构的整体磁盘组织。我们需要做的第一件事是将磁盘划分为块；简单的文件系统只使用一种块大小，这正是我们在这里要做的。让我们选择一个常用的大小 4 KB.

因此，我们对构建文件系统的磁盘分区的看法很简单：一系列块，每个块大小为 4 KB。这些块的地址从 0 到 N - 1，位于大小为 N 个 4 KB 块的分区中。假设我们有一个非常小的磁盘，只有 64 个块：
![[Pasted image 20241203114357.png]]

现在让我们考虑一下我们需要在这些块中存储什么来构建文件系统。当然，首先想到的是用户数据。事实上，任何文件系统中的大部分空间都是（也应该是）用户数据。我们将用于用户数据的磁盘区域称为数据区域，同样为了简单起见，为这些块保留磁盘的固定部分，比如磁盘上 64 个块中的最后 56 个：

![[Pasted image 20241203114442.png]]

正如我们在上一章中了解到的（一点点），文件系统必须跟踪有关每个文件的信息。此信息是元数据的关键部分，并跟踪哪些数据块（在数据区域中）组成文件、文件的大小、其所有者和访问权限、访问和修改时间以及其他类似类型的信息。为了存储这些信息，文件系统通常具有一种称为 inode 的结构。

为了容纳 inode，我们还需要在磁盘上为它们保留一些空间。我们将磁盘的这一部分称为 **inode table**，它只是保存磁盘上 inode 的数组。因此，我们的磁盘映像现在看起来像这张图片，假设我们将 64 个块中的 5 个用于 inode（在图中用 I 表示）：

![[Pasted image 20241203114634.png]]

这里需要注意的是，inode 通常不会那么大，例如 128 或 256 字节。假设每个 inode 有 256 字节，则 4 KB 块可以容纳 16 个 inode，而我们上面的文件系统总共包含 80 个 inode。在我们基于一个微小的 64 块分区构建的简单文件系统中，这个数字代表了我们的文件系统中可以容纳的最大文件数量；但是，请注意，基于更大磁盘构建的相同文件系统可以简单地分配更大的 inode 表，从而容纳更多文件。

到目前为止，我们的文件系统有数据块 (D) 和 inode (I)，但仍缺少一些东西。您可能已经猜到了，仍然需要的一个主要组件是某种方法来跟踪 inode 或数据块是空闲的还是已分配的。因此，这种**allocation structures** 是任何文件系统中必不可少的元素。

当然，许多分配跟踪方法都是可行的。例如，我们可以使用指向第一个空闲块的 free list，然后指向下一个空闲块，依此类推。我们选择一种简单而流行的结构，称为位图，一个用于数据区域（数据位图），一个用于 inode 表（inode 位图）。A bitmap is a simple structure: each bit is used to indicate whether the corresponding object/block is free (0) or in-use(1). And thus our new on-disk layout, with an inode bitmap (i) and a data bitmap (d):
![[Pasted image 20241203115040.png]]

您可能会注意到，为这些位图使用整个 4 KB 块有点过头了；这样的位图可以跟踪 32K 对象是否已分配，而我们只有 80 个 inode 和 56 个数据块。不过，为了简单起见，我们只为每个位图使用整个 4 KB 块。

细心的读者（即仍然清醒的读者）可能已经注意到，在我们非常简单的文件系统的磁盘结构设计中还剩下一个块。我们将其保留给超级块，在下图中用 S 表示。超级块包含有关此特定文件系统的信息，例如，文件系统中有多少个 inode 和数据块（在本例中分别为 80 个和 56 个）、inode 表的起始位置（块 3）等等。它还可能包含某种类型的魔力数字来识别文件系统类型（在本例中为 vsfs）。

![[Pasted image 20241203115213.png]]

Thus, when mounting a file system, the OS will read the superblock first, to initialize various parameters, and then attach the volume to the file-system tree. When files within the volume are accessed, the system will thus know exactly where to look for the needed on-disk structures.

# 40.3 File Organization: The Inode
每个 inode 都隐式地由一个数字（称为 i 号）引用，我们之前将其称为文件的低级名称。在 vsfs（和其他简单文件系统）中，给定一个 i 号，您应该能够直接计算出相应 inode 在磁盘上的位置。例如，以上面的 vsfs 的 inode 表为例：大小为 20KB（五个 4KB 块），因此由 80 个 inode 组成（假设每个 inode 为 256 字节）；进一步假设 inode 区域从 12KB 开始（即超级块从0KB 开始，inode 位图位于地址 4KB，数据位图位于 8KB，因此 inode 表紧随其后）。因此，在 vsfs 中，我们有以下文件系统分区开头的布局（特写视图）：
![[Pasted image 20241203115719.png]]

要读取 inode 编号 32，文件系统将首先计算 inode 区域的偏移量（32 · sizeof(inode) 或 8192），将其添加到磁盘上 inode 表的起始地址（inodeStartAddr = 12KB），从而得到所需 inode 块的正确字节地址：20KB. 回想一下，磁盘不是字节寻址的，而是由大量可寻址扇区组成，通常为 512 字节。因此，要获取包含 inode 32 的 inode 块，文件系统将发出对扇区 20×1024
/ 512 或 40 的读取，以获取所需的 inode 块。更一般地说，inode 块的扇区地址扇区可以按如下方式计算：
![[Pasted image 20241203115915.png]]

每个 inode 中几乎包含您需要的有关文件的所有信息：文件类型（例如，常规文件、目录等）、文件大小、分配给它的块数、保护信息（例如，谁拥有该文件以及谁可以访问它）、一些时间信息（包括文件的创建、修改或上次访问时间），以及有关其数据块在磁盘上的位置的信息（例如，某种指针）。我们将有关文件的所有此类信息称为元数据；事实上，文件系统内任何不是纯用户数据的信息通常都被称为元数据。图 40.11 显示了 ext2 中的一个示例 inode。

![[Pasted image 20241203120008.png]]

在设计 inode 时，最重要的决定之一是它如何引用数据块的位置。一种简单的方法是在 inode 内设置一个或多个直接指针（磁盘地址）；每个指针指向属于文件的一个磁盘块。这种方法有局限性：例如，如果你想要一个非常大的文件（例如，大于块大小乘以 inode 中的直接指针数量），你就没那么幸运了。

## The Multi-Level Index
为了支持更大的文件，文件系统设计人员不得不在 inode 中引入不同的结构。一个常见的想法是使用一个称为间接指针的特殊指针。它不是指向包含用户数据的块，而是指向包含更多指针的块，每个指针都指向用户数据。因此，inode 可能具有一些固定数量的直接指针（例如 12 个）和一个间接指针。如果文件变得足够大，则会分配一个间接块（来自磁盘的数据块区域），并将 inode 的间接指针槽设置为指向它。假设 4 KB 块和 4 字节磁盘地址，则会增加另外 1024 个指针；文件可以增长到 (12 + 1024) · 4K 或 4144KB。

毫不奇怪，在这种方法中，您可能希望支持更大的文件。为此，只需向 inode 添加另一个指针：双重间接指针。此指针指向一个包含指向间接块的指针的块，每个间接块都包含指向数据块的指针。因此，双重间接块增加了使用额外的 1024 · 1024 或 100 万个 4KB 块来扩展文件的可能性，换句话说，支持大小超过 4GB 的文件。不过，您可能想要更多，我们敢打赌您知道这是什么方向：三重间接指针。

许多文件系统都使用多级索引，包括常用的文件系统，例如 Linux ext2 和 ext3、NetApp 的 WAFL，以及原始的 UNIX 文件系统。其他文件系统，包括 SGI XFS 和 Linux ext4，使用范围而不是简单指针；

# 40.4 Directory Organization
在 vsfs 中（与许多文件系统一样），目录具有简单的组织；目录基本上只包含（条目名称、inode 编号）对的列表。对于给定目录中的每个文件或目录，目录的数据块中都有一个字符串
和一个数字。对于每个字符串，也可能有一个长度（假设名称大小可变）。

For example, assume a directory `dir` (inode number 5) has three files in it (foo, bar, and foobar_is_a_pretty_longname), with inode number 12, 13, and 24 respectively. The on-disk data for `dir` might look like:

![[Pasted image 20241203121037.png]]

在此示例中，每个条目都有一个 inode 编号、记录长度（名称的总字节数加上任何剩余空间）、字符串长度（名称的实际长度），最后是条目的名称。

删除文件（例如，调用 unlink()）可能会在目录中间留下空白，因此也应该有某种方法来标记该空白（例如，使用保留的 inode 编号，例如零）。这种删除是使用记录长度的原因之一：新条目可能会重用旧的、更大的条目，因此其中会有额外的空间。

您可能想知道目录究竟存储在哪里。通常，文件系统将目录视为一种特殊类型的文件。因此，目录在 inode 表中的某个位置有一个 inode（inode 的类型字段标记为“目录”而不是“常规文件”）。目录具有由 inode 指向的数据块（可能还有间接块）；这些数据块位于我们简单文件系统的数据块区域中。因此，我们的磁盘结构保持不变。

我们还应该再次注意，这个简单的目录条目线性列表并不是存储此类信息的唯一方法。和以前一样，任何数据结构都是可能的。例如，XFS 以 B 树形式存储目录，使文件创建操作（必须确保在创建文件名之前未使用过）比必须完整扫描的简单列表系统更快。

# 40.5 Free Space Management
文件系统必须跟踪哪些 inode 和数据块是空闲的，哪些不是，这样当分配新文件或目录时，它就可以为其找到空间。因此，空闲空间管理对所有文件系统都很重要。在 vsfs 中，我们有两个简单的位图用于此任务。

例如，当我们创建一个文件时，我们必须为该文件分配一个 inode。因此，文件系统将在位图中搜索可用的 inode，并将其分配给该文件；文件系统必须将 inode 标记为已使用（标记为 1），并最终使用正确的信息更新磁盘上的位图。分配数据块时也会发生一组类似的活动。

在为新文件分配数据块时，可能还需要考虑其他一些因素。例如，某些 Linux 文件系统（如 ext2 和 ext3）会在创建新文件并需要数据块时寻找一系列空闲的块（例如 8 个）；通过找到这样的空闲块序列，然后将它们分配给新创建的文件，文件系统可以保证文件的一部分在磁盘上是连续的，从而提高性能。因此，这种预分配策略是分配数据块空间时常用的启发式方法。

# 40.6 Access Paths: Reading and Writing
现在我们已经了解了文件和目录如何存储在磁盘上，我们应该能够在读取或写入文件的过程中遵循操作流程。因此，了解此访问路径上发生的事情是了解文件系统工作原理的第二个关键；注意！

对于以下示例，我们假设文件系统已挂载，==**因此超级块已在内存中**。其他所有内容（即 inode、目录）仍在磁盘上。==

## Reading A File From Disk
In this simple example, let us first assume that you want to open a file (e.g., /foo/bar), read it, and then close it. For this simple example, let's assume the file is just 12KB in size (i.e., 3 blocks).

When you issue an open ("/foo/bar", O_RDONLY) call, the file system first needs to find the inode for the file `bar`, to obtain some basic information about the file (permission information, file size, etc.). To do so, the file system must be able to find the inode, but all it has right now is the full pathname. The file system must **traverse** the pathname and thus locate the desired inode.

All traversals begin at the root of the file system, in the **root directory** which is simply called /. Thus, the first thing the FS will read from disk is the inode of the root directory. But where is this inode? To find an inode, we must know its i-number. Usually, we find the i-number of a file or directory in ites parent directory; the root has no parent. Thus, the root inode number be "well known"; the FS must know what it is when the file system is mounted. In most UNIX FS, the root inode number is 2. Thus, to begin the process, the FS reads in the block that contains inode number 2 (the first inode block).

一旦读入了 inode，FS 就可以查看其中的内容，以找到指向数据块的指针，其中包含根目录的内容。因此，FS 将使用这些磁盘上的指针来读取目录，在本例中是查找 foo 的条目。通过读取一个或多个目录数据块，它将找到 foo 的条目；一旦找到，FS 还将找到接下来需要的 foo 的 inode 编号（假设为 44）。

下一步是递归遍历路径名，直到找到所需的 inode。在此示例中，FS 读取包含 foo 的 inode 的块，然后读取其目录数据，最终找到 bar 的 inode 编号。The final step of `open()` is to read `bar`'s inode into memory; the FS then does a final permissions check, allocate a file descriptor for this process in the per-process open-file table, and returns it to the user.

打开后，程序便可以发出 read() 系统调用来读取文件。因此，第一次读取（在偏移量 0 处，除非已调用 lseek()）将读取文件的第一个块，并查询 inode 以查找此类块的位置；它还可能使用新的上次访问时间更新 inode。读取将进一步更新此文件描述符的内存中打开文件表，更新文件偏移量，以便下一次读取将读取第二个文件块，等等.

在某个时刻，文件将被关闭。这里要做的工作要少得多；显然，文件描述符应该被释放，但现在，这就是 FS 真正需要做的一切。没有发生磁盘 I/O。

整个过程的描述如图 40.3（第 11 页）所示；图中时间向下增加。在图中，打开会导致发生大量读取，以便最终找到文件的 inode。之后，读取每个块需要文件系统首先查阅 inode，然后读取块，然后使用写入更新 inode 的最后访问时间字段。

还要注意，打开生成的 I/O 量与路径名的长度成正比。对于路径中的每个附加目录，我们必须读取其 inode 及其数据。如果存在大型目录，情况会变得更糟；
![[Pasted image 20241203122643.png]]

## Writing A File To Disk
Writing to a file is a similar process. First, the file must be opened. Then, the application can issue `write()` calls to update the file with new contents. Finally, the file is closed.

与读取不同，写入文件也可能分配一个块（例如，除非该块被覆盖）。在写出新文件时，每次写入不仅必须将数据写入磁盘，还必须首先决定将哪个块分配给文件，从而相应地更新磁盘的其他结构（例如，数据位图和 inode）。因此，每次写入文件在逻辑上都会生成五个 I/O：一个用于读取数据位图（然后更新以将新分配的块标记为已使用），一个用于写入位图（以将其新状态反映到磁盘），另外两个用于读取然后写入 inode（使用新块的位置进行更新），最后一个用于写入实际块本身。

当考虑文件创建等简单而常见的操作时，写入流量的数量甚至会更糟。要创建文件，文件系统不仅必须分配一个 inode，还必须在包含新文件的目录中分配空间。这样做的总 I/O 流量相当高：一次读取 inode 位图（以查找空闲的 inode），一次写入 inode 位图（以将其标记为已分配），一次写入新 inode 本身（以初始化它），一次写入目录的数据（以将文件的高级名称链接到其 inode 编号），以及一次读取和写入目录 inode 以更新它。如果目录需要增长以容纳新条目，则还需要额外的 I/O（即，数据位图和新目录块）。所有这些只是为了创建一个文件！

Let's look at a specific example, where the file `/foo/bar` is created, and three blocks are written to it.

在图中，对磁盘的读写操作按导致它们发生的系统调用分组，它们发生的粗略顺序从图的顶部到底部。您可以看到创建文件需要做多少工作：在本例中需要 10 次 I/O，遍历路径名，然后最终创建文件。您还可以看到每次分配写入都需要 5 次 I/O：一对用于读取和更新 inode，另一对用于读取和更新数据位图，最后是数据本身的写入。文件系统如何以合理的效率完成所有这些工作？


![[Pasted image 20241203131633.png]]

How to reduce  FS I/O costs? Even the simplest of operations like opening, reading, or writing a file incurs a huge number of I/O operations, scattered over the disk. What can a FS do to reduce the high costs of doing so many I/Os?

# 40.7 Caching and Buffering
如上例所示，读取和写入文件的成本可能很高，会导致磁盘（速度较慢）出现大量 I/O 操作。为了解决这个显然会造成巨大性能问题的问题，大多数文件系统都会积极使用系统内存 (DRAM) 来缓存重要数据块。

想象一下上面的打开示例：如果没有缓存，每个打开的文件都需要对目录层次结构中的每个级别进行至少两次读取（一次读取相关目录的 inode，至少一次读取其数据）。如果路径名很长（例如，/1/2/3/ ... /100/file.txt），文件系统实际上需要执行数百次读取才能打开文件！

因此，早期的文件系统引入了固定大小的缓存来保存常用的块。正如我们在虚拟内存的讨论中所说，LRU 等策略和不同的变体将决定哪些块保留在缓存中。这个固定大小的缓存通常会在启动时分配，大约占总内存的 10%。

然而，这种静态的内存分区可能会造成浪费；如果文件系统在某个时间点不需要 10% 的内存怎么办？使用上述固定大小的方法，文件缓存中未使用的页面无法重新用于其他用途，因此会被浪费。

相比之下，现代系统采用动态分区方法。具体来说，许多现代操作系统将虚拟内存页面和文件系统页面集成到统一的页面缓存中。这样，可以在虚拟内存和文件系统之间更灵活地分配内存，具体取决于在给定时间内哪个需要更多内存。

现在想象一下使用缓存打开文件的示例。第一次打开可能会产生大量 I/O 流量来读取目录 inode 和数据，但同一文件（或同一目录中的文件）的后续文件打开将主要命中缓存，因此不需要 I/O。

我们还需要考虑缓存对写入的影响。虽然使用足够大的缓存可以完全避免读取 I/O，但写入流量必须进入磁盘才能持久。Thus, a cache does not serve as the same kind of filter on write traffic that it does for reads. That said, **write buffering** certainly has a number of performance benefits. First, by delaying writes, the file system can batch some updates into a smaller set of I/Os;  例如，如果在创建一个文件时更新 inode 位图，然后在创建另一个文件时稍后更新，则文件系统通过延迟第一次更新后的写入来节省 I/O。其次，通过在内存中缓冲大量写入，系统可以安排后续 I/O，从而提高性能。最后，通过延迟写入可以完全避免一些写入；例如，如果应用程序创建了一个文件然后删除了它，则延迟写入以反映文件创建到磁盘可以完全避免这些写入。在这种情况下，懒惰（将数据块写入磁盘）是一种美德。

出于上述原因，大多数现代文件系统都会将写入操作缓冲在内存中 5 到 30 秒，这又代表着另一种权衡：如果系统在更新操作传播到磁盘之前崩溃，则更新操作将会丢失；但是，通过将写入操作保留在内存中更长时间，可以通过批处理、调度甚至避免写入来提高性能。

某些应用程序（例如数据库）不喜欢这种权衡。因此，为了避免由于写入缓冲而导致意外的数据丢失，它们只需强制写入磁盘，方法是调用 `fsync()`、使用绕过缓存的`direct I/O` 接口，或者使用`raw disk`接口并完全避免使用文件系统。虽然大多数应用程序都接受文件系统的权衡，但如果默认设置不令人满意，则有足够的控制措施让系统执行您想要的操作。


## 提示：了解静态与动态分区
在将资源分配给不同的客户端/用户时，您可以使用静态分区或动态分区。静态方法只是将资源一次性划分为固定比例；例如，如果有两个可能的内存用户，您可以将固定比例的内存分配给一个用户，将其余部分分配给另一个用户。动态方法更灵活，会随时间提供不同数量的资源；例如，一个用户可能在一段时间内获得更高百分比的磁盘带宽，但随后系统可能会切换并决定为不同的用户提供更大比例的可用磁盘带宽。每种方法都有其优点。静态分区确保每个用户都能获得一定比例的资源，通常可提供更可预测的性能，并且通常更易于实施。动态分区可以实现更好的利用率（通过让资源匮乏的用户消耗原本闲置的资源），但实施起来可能更复杂，并且会导致那些闲置资源被其他用户消耗，然后在需要时需要很长时间才能回收的用户的性能下降。通常情况下，没有最好的方法；相反，你应该考虑手头的问题，并决定哪种方法最合适。事实上，你难道不应该一直这样做吗？


# 40.8 Summary
我们已经了解了构建文件系统所需的基本机制。每个文件都需要一些信息（元数据），这些信息通常存储在称为 inode 的结构中。Directories are just a specific type of file that store name -> inode-number mappings. And other structures are needed too; for example, file systems often use a structure such as a bitmap to track which inodes or data blocks are free or allocated.

文件系统设计的一大优点是它的自由度；我们在接下来的章节中探讨的文件系统都利用这种自由度来优化文件系统的某些方面。显然，还有很多策略决策我们还没有探索。例如，当创建一个新文件时，它应该放在磁盘的什么位置？这项政策和其他政策也将成为未来章节的主题。或者会是吗？







