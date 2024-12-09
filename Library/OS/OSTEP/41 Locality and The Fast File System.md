1Old Unix file system, and it was really simple. Basically, its data structures looked like this on the disk:
![[Pasted image 20241204184322.png]]
The super block (S) contained information about the entire file system: how big the volume is, how many inodes there are, a pointer to the head of a free list of blocks, and so forth. The inode region of the disk contained all the inodes for the file system.

# 41.1 The Problem: Poor Performance
Performance started off bad and got worse over time, to the point where the file system was delivering only 2% of overall disk bandwidth.

主要问题是，旧 UNIX 文件系统将磁盘视为随机存取存储器；数据分散在各处，而没有考虑到保存数据的介质是磁盘，因此会产生实际且昂贵的定位成本。例如，文件的数据块通常距离其 inode 非常远，因此每当先读取 inode 然后读取文件的数据块（一种非常常见的操作）时，都会引发昂贵的寻道。

Worse, the file system would end up getting quite fragmented, as the free space was not carefully managed. 空闲列表最终会指向分散在磁盘上的一堆块，当文件被分配时，它们会直接占用下一个空闲块。结果是，逻辑上连续的文件需要通过磁盘来回访问，从而大大降低了性能。

One other problem: the original block size was too small (512 bytes). Thus, transferring data from the disk was inherently inefficient.

## the crux
How to organize on-disk data to improve performance? How can we organize file system data structures so as to improve performance? What types of allocation policies do we need on top of those data strutures? How do we make the file system "disk aware"?

# 41.2 FFS: Disk Awareness is the solution
伯克利的一个小组决定构建一个更好、更快的文件系统，他们巧妙地将其称为快速文件系统 (FFS)。他们的想法是将文件系统结构和分配策略设计为“磁盘感知”，从而提高性能，而这正是他们所做的。因此，FFS 开创了文件系统研究的新时代；通过保留与文件系统相同的接口（相同的 API，包括 open()、read()、write()、close() 和其他文件系统调用），但更改内部实现，作者为新文件系统的构建铺平了道路，这项工作一直持续到今天。几乎所有现代文件系统都遵循现有接口（从而保持与应用程序的兼容性），同时出于性能、可靠性或其他原因更改其内部结构。

# 41.3 Organizing Structure: The Cylinder Group
第一步是改变磁盘上的结构。FFS 将磁盘划分为多个柱面组。单个柱面是硬盘驱动器不同表面上的一组磁道，这些磁道与驱动器中心的距离相同；它之所以被称为柱面，是因为它与所谓的几何形状非常相似。FFS 将 N 个连续的柱面聚合为一个组，因此整个磁盘可以看作是一组柱面组。这是一个简单的例子，显示了一个有六个盘片的驱动器的四个最外层磁道，以及一个由三个柱面组成的柱面组：
![[Pasted image 20241205141548.png]]

请注意，现代驱动器不会导出足够的信息，以便文件系统真正了解特定柱面是否正在使用；如前所述，磁盘导出块的逻辑地址空间，并向客户端隐藏其几何形状的详细信息。因此，现代文件系统（如 Linux ext2、ext3 和 ext4）将驱动器组织成块组，每个块组只是磁盘地址空间的连续部分。下图说明了每 8 个块被组织成一个不同的块组的示例（请注意，实际组将由更多块组成）：
![[Pasted image 20241205141739.png]]
无论您称它们为柱面组还是块组，这些组都是 FFS 用来提高性能的核心机制。至关重要的是，通过将两个文件放在同一个组中，FFS 可以确保依次访问不会导致在磁盘上进行长时间寻道。

要使用这些组来存储文件和目录，FFS 需要能够将文件和目录放入一个组中，并在其中跟踪有关它们的所有必要信息。为此，FFS 包括您可能期望文件系统在每个组中具有的所有结构，例如，用于 inode、数据块的空间，以及一些用于跟踪每个结构是否已分配或空闲的结构。以下是 FFS 在单个柱面组中保存的内容的描述：
![[Pasted image 20241205141842.png]]
FFS keeps a copy of the super block (S) in each group for reliability reasons. The super block is needed to mount the file system; by keeping multiple copies, if one copy becomes corrupt, you can still mount and access the file system by using a working replica.

Within each group, FFS needs to track whether the inodes and data blocks of the group are allocated. A per-group inode bitmap (ib) and data bitmap (db) serve this role for inodes and data blocks in each group. 

Finally, the inode and data block regions are just like those in the previous very-simple file system (VSFS). Most of each cylinder group, as usual, is comprised of data blocks.

## 附言：FFS 文件创建
作为示例，考虑在创建文件时必须更新哪些数据结构；在此示例中，假设用户创建了一个新文件 /foo/bar.txt，并且该文件的长度为一个块（4KB）。该文件是新的，因此需要一个新的 inode；因此，inode 位图和新分配的 inode 都将写入磁盘。该文件中还有数据，因此也必须分配；因此，数据位图和数据块将（最终）写入磁盘。因此，将对当前柱面组进行至少四次写入（请记住，这些写入可能会在内存中缓冲一段时间后再进行）。但这还不是全部！特别是，在创建新文件时，还必须将文件放置在
文件系统层次结构中，即必须更新目录。具体而言，必须更新父目录 foo 以添加 bar.txt 的条目；此更新可能适合 foo 的现有数据块，或者需要分配一个新块（带有相关数据位图）。还必须更新 foo 的 inode，以反映目录的新长度以及更新时间字段（例如上次修改时间）。总的来说，创建一个新文件需要做很多工作！也许下次你这样做时，你应该更加感激，或者至少惊讶于它运行得如此顺利。

# 41.4 Policies: How to Allocate Files and Directories
有了这个组结构，FFS 现在必须决定如何将文件和目录以及相关元数据放置在磁盘上以提高性能。基本原则很简单：将相关内容放在一起（其推论是将不相关内容分开）。

因此，为了遵循这一原则，FFS 必须确定什么是“相关的”，并将其放在同一个块组中；相反，不相关的项目应该放在不同的块组中。为了实现这一目标，FFS 使用了一些简单的放置启发式方法。

首先是目录的放置。FFS 采用一种简单的方法：找到分配目录数量较少（以平衡组之间的目录）且空闲 inode 数量较多（以便随后能够分配一堆文件）的柱面组，并将目录数据和 inode 放在该组中。当然，这里可以使用其他启发式方法（例如，考虑空闲数据块的数量）。

对于文件，FFS 会执行两项操作。首先，它确保（在一般情况下）将文件的数据块分配到与其 inode 相同的组中，从而防止 inode 和数据之间的长时间寻道（如在旧文件系统中一样）。其次，它将同一目录中的所有文件放置在它们所在目录的磁柱组中。因此，如果用户创建了四个文件 /a/b、/a/c、/a/d 和 b/f，FFS 会尝试将前三个文件放在彼此附近（同一组），将第四个文件放在较远的位置（其他组中）。

# 41.8 Summary
FFS 的推出是文件系统历史上的一个分水岭，因为它明确表明文件管理问题是操作系统中最有趣的问题之一，并展示了如何开始处理最重要的设备——硬盘。从那时起，数百种新的文件系统已经开发出来，但今天仍然有许多文件系统从 FFS 中汲取灵感（例如，Linux ext2 和 ext3 显然是其思想的继承者）。当然，所有现代系统都体现了 FFS 的主要教训：将磁盘视为磁盘。