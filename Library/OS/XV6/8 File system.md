The xv6 file system addresses several challenges:
- The file system needs on-disk data structures to represent the tree of named directories and files, to record the identities of the blocks that hold each file's content, and to record which areas of the disk are free.
- The file system must support _crash recovery_. That is, if a crash (e.g., power failure) occurs, the file system must still work after a restart. The risk is that a crash might interrupt a sequence of updates and leave inconsistent on-disk data structures.
- Different processes may operate on the file system at the same time, so the file-system code must coordinate to maintain invariants.
- Accessing a disk is orders of magnitude slower that accessing memory, so the file system must maintain an in-memory cache of popular blocks.

# 8.1 Overview
Seven layers, shown in Figure 8.1.
![[Pasted image 20241226190546.png]]
The disk layer reads and writes blocks on an virtio hard drive. The buffer cache layer caches disk blocks and synchronizes access to them, making sure that only one kernel process at a time can modify the data stored in any particular block. The logging layer allows higher layers to wrap updates to several blocks in a _transaction_, and ensures that the blocks are updated atomically in the face of crashes (i.e., all of them are updated or none). The inode layer provides individual files, each represented as an _inode_ with a unique i-number and some blocks holding the file's data. The directory layer implements each directory as a special kind of inode whose content is a sequence of directory entries, each of which contains a file's name and i-number. The pathname layer provides hierarchical path names like `/usr/rtm/xv6/fs.c`, and resolves them with recursive lookup. The file descriptor layer abstracts many Unix resources (e.g., pipes, devices, files, etc.) using the file system interface, simplifying the lives of application programmers.

Disk hardware presents the data on the disk as a number sequence of 512-byte _blocks_ (also called _sectors_). 操作系统用于其文件系统的块大小可能与磁盘使用的扇区大小不同，但通常块大小是扇区大小的倍数。xv6 将其已读入内存的块的副本保存在 `struct buf` (kernel/buf.h:1) 类型的对象中。存储在此结构中的数据有时与磁盘不同步：它可能尚未从磁盘读入（磁盘正在处理它但尚未返回扇区的内容），或者它可能已被软件更新但尚未写入磁盘。

Xv6 divides the disk into several sections. The file system does not use block 0 (it holds the boot sector). Block 1 is called the _superblock_; it contains metadata about the file system. Blocks starting at 2 hold the log. After the log are the inodes, with multiple inodes per block. After those come bitmap blocks tracking which data blocks are in use. The remaining blocks are data blocks; each is marked free in the bitmap block, or holds content for a file or directory. The superblocks is filled in by a separate program, called `mkfs`, which builds an initial file system.
![[Pasted image 20241226192015.png]]
- block 0: 启动区域，文件系统不会使用，包含了操作系统启动所需要的代码
- blcok 1: _superblock_，保存着文件系统的元数据、一些用于构建初始操作系统的代码（称为 `mkfs`）
	- 元数据
		- 文件系统的大小（多少块）
		- 数据有多少块
		- inode 有多少个
		- log 占多少块
- block 2-31：log block，大小为 LOGSIZE+1=(MAXOPBLOCKS * 3)+1=31
- block 32-44: inode，一个inode的大小为64字节，一个block的大小为1024字节，因此block32为inode 1-16，block33为inode 17-32
- block 45 bitmap block，用来跟踪哪些block是在使用
- 最后从block 46开始是data block，要么是在bitmap中被标记为空闲状态，要么存储了文件/文件夹的内容

# 8.2 Buffer cache layer
- 两个目标
    - 同步磁盘块，保证每个磁盘块在内存中最多只有一个拷贝，保证每一个磁盘块的拷贝只能被一个内核线程使用
    - 缓存 popular blocks，减少访问磁盘导致的开销
- 对上提供的接口如下
    - bread()：获取一个磁盘块的拷贝到内存中
    - bwrite()：将缓存写入磁盘的对应块中
    - brelse()：释放缓存块（**用完必须得释放**）
- 通过给每一个 buffer 分配一个 sleeplock 的方式实现一个磁盘块的拷贝只能被一个内核线程使用
    - bread() 返回一个带锁的 buffer
    - brelse() 释放锁
- 当需要的磁盘块没有读入内存的时候，通过 LRU 的方式回收一个 buffer，并从磁盘中将对应块读入内存
    - 通过 `virtio_disk_rw(b, 0)` 实现

`bread` and `bwrite`; the former obtains a `buf` containing a copy of a block which can be read or modified in memory, and the latter writes a modified buffer to the appropriate block on the disk. A kernel thread must release a buffer by calling `brelse` when it is done with it. The buffer cache uses a per-buffer sleep-lock to ensure that only one thread at a time uses each other (and thus each disk block); `bread` returns a locked buffer, and `brelse` releases the lock.

块缓冲有固定数量的缓冲区，这意味着如果文件系统请求一个不在缓冲中的块，必须换出一个已经使用的缓冲区。这里的置换策略是 LRU。优先淘汰最久没有被使用的数据。

# 8.3 Code: Buffer cache
- 一共有 30 个 buffer
    - `NBUF=(MAXOPBLOCKS*3)=30`

- 通过 `binit()` 将，所有的 buffer 保存为一个静态的数组
    - 组织成一个双向循环链表（为了实现 LRU 的替换算法）
    - 为了方便组织，这个双向链表带一个空的虚拟头结点
	
- 每一个 buffer 的数据结构如下
    - valid 字段表示这个 buffer 是否对应磁盘上的某一个块（1 表示有对应）
    - disk 字段表示是否将 buffer 中的信息写回了磁盘上（1 表示修改了未写回）
![[Pasted image 20241227223156.png]]
- bread() 函数调用 bget() 获取一个 buffer
    - 如果返回一个 `valid=1` 的 buffer，说明待读取的 buffer 已经在内存中了
    - 如果返回一个 `valid=0` 的 buffer，说明待读取的 buffer 尚未进入内存，通过调用 `virtio_disk_rw(b, 0)` 从磁盘中将其读入内存（buffer 中）

- bget() 的逻辑如下
    - 首先顺序扫描整个链表，看需要读取的内容是否被缓存，若缓存了，直接返回
    - 若未缓存，从链表开始往前扫描（LRU），找到一个空闲的 buffer（refcnt=0） 返回
        - 注意此时需要把 valid 项置为 0，表明需要从磁盘中读取具体的内容
    - 如果还没找到，则报错 `panic()`
    - 注意返回的 buffer 是带 sleep-lock 
- bget() 的整个过程都是持有锁 `bcache.lock` 的，因此保证了每一个磁盘块在 buffer 中最多只会有一个拷贝
- 如果修改了 buffer 中的内容，需要调用 bwrite() 将内容写回磁盘，内部调用 `virtio_disk_rw(b, 1)` 实现
- 最终需要调用 brelse() 进行释放缓存块，将 refcnt 减 1，同时得释放 buffer 上的锁
    - 如果 refcnt 为 0，将这个 buffer 插入到链表头的后面
        - 这就保证了 LRU 的实现，在逆向查找，找到的第一个空闲块一定是最老的

`bget` scans the buffer list for a buffer with the given device and sector numbers. If there is such a buffer, `bget` acquires the sleep-lock for the buffer. `bget` then returns the locked buffer.

It is safe for `bget` to acquire the buffer's sleep-lock outside of the `bcache.lock` critical section, since the non-zero `b->refcnt` prevents the buffer from being re-used for a different disk block. The sleep-lock protects reads and writes of the block's buffered content, while the `bcache.lock` protects information about which blocks are cached.

If all the buffers are busy, then too many processes are simultaneously executing file system calls; `bget` panics. A more graceful response might be to sleep until a buffer became free, though there would then be a possibility of deadlock.

Once `bread` has read the disk (if needed) and returned the buffer to its caller, the caller has exclusive use of the buffer and can read or write the data bytes. If the caller does modify the buffer, it must call `bwrite` to write the changed data to disk before releasing the buffer. `bwrite` calls `virtio_disk_rw` to talk to the disk hardware.

When the caller is done with a buffer, it must call `brelse` to release it. 

# 8.4 Logging layer
- 当需要向磁盘写入内容的时候，xv6 先向磁盘上写一个日志记录，当把所有的写操作都计入日志之后，写一个 commit 表示日志记录完整，此时再进行系统调用将内容写入磁盘，当内容都写完之后，在磁盘上消去这个日志信息
- 系统崩溃重启之后的流程如下
    - 检查日志记录，如果有完整的操作，那么就执行，如果操作不完整，则直接忽略
Crash recovery. The problem arises because many file-system operations involve multiple writess to the disk, and a crash after a subset of the writes may leave the on-disk file system in an inconsistent state.

Xv6 solves the problem of crashes during file-system operations with a simple form of logging. An xv6 system call does not directly write the on-disk file system data structures. Instead, it places a description of all the disk writes it wishes to make in a _log_ on the disk. Once the system call has logged all of its writes, it writes a special _commit_ record to the disk indicating that the log contains a complete operation. At that point the system call copies the writes to the on-disk file system data structures. After those writes have completed, the system call erases the log on disk.

The log makes operations atomic with respect to crashes: after recovery, either all of the operation's writes appear on the disk, or none of them appear.

# 8.5 Log design
日志存在于磁盘末端已知的固定区域。它包含了一个起始块，紧接着一连串的数据块。起始块包含了一个扇区号的数组，每一个对应于日志中的数据块，起始块还包含了日志数据块的计数。xv6 在提交后修改日志的起始块，而不是之前，并且在将日志中的数据块都拷贝到文件系统之后将数据块计数清0。提交之后，清0之前的崩溃就会导致一个非0的计数值。

每一个系统调用都可能包含一个必须从头到尾原子完成的写操作序列，我们称这样的一个序列为一个会话，虽然他比数据库中的会话要简单得多。任何时候只能有一个进程在一个会话之中，其他进程必须等待当前会话中的进程结束。因此同一时刻日志最多只记录一次会话。

每个系统调用的代码都表明了写入序列的开始和结束，这些写入序列在崩溃时必须是原子的。为了允许不同的进程并发执行文件系统操作，日志系统可以将多个系统调用的写入累积到一个事务中。因此，一次提交可能涉及多个完整系统调用的写入。为了避免将系统调用拆分到事务中，日志系统仅在没有文件系统系统调用正在进行时才提交。

将多个事务一起提交的想法称为 _group commit_。组提交减少了磁盘操作的数量，因为它将提交的固定成本分摊到多个操作上。组提交还会同时向磁盘系统提供更多并发写入，也许允许磁盘在单个磁盘旋转期间写入所有写入。xv6 的 virtio 驱动程序不支持这种批处理，但 xv6 的文件系统设计允许这样做。

Xv6 在磁盘上专门留出固定数量的空间来保存日志。事务中系统调用写入的总块数必须适合该空间。这有两个后果。任何单个系统调用都不能写入比日志空间更多的不同块。这对大多数系统调用来说都不是问题，但其中两个系统调用可能会写入许多块：write 和 unlink。大文件写入可能会写入许多数据块和许多位图块以及一个 inode 块；unlink 大文件可能会写入许多位图块和一个 inode。Xv6 的写入系统调用将大写入分解为多个适合日志的较小写入，而 unlink 不会导致问题，因为实际上 xv6 文件系统仅使用一个位图块。日志空间有限的另一个后果是，除非确定系统调用的写入适合日志中剩余的空间，否则日志系统无法允许系统调用启动。

# 8.6 Code: logging
A typical use of the log in a system call looks like this:
![[Pasted image 20241227215631.png]]
`begin_op()` waits until the logging system is not currently committing, 会一直等到它独占了日志的使用权后返回。and until there is enough unreserved log space to hold the writes from this call.
`log.outstanding` counts the number of system calls that have reserved log space; the total reserved space is `log.outstanding` times `MAXOPBLOCKS`. Incrementing `log.outstanding` both reserves space and prevents a commit from occuring during this system call. 代码保守地假设每个系统调用可能会写入最多 MAXOPBLOCKS 个不同的块。

#### begin_op()
- 等待，直到 log 系统没有 committing 而且有充足的 log blocks
    - xv6 认为每个系统调用只会使用 MAXOPBLOCKS（10） 个 block
- 满足条件后，系统调用数 +1 并返回
    - log.outstanding 记录使用 log 系统的系统调用数

`log_write` acts as a proxy for `bwrite`. 它将块的扇区号记录在内存中，并在磁盘日志中为其保留一个位置，并将缓冲区固定在块缓存中，以防止块缓存将其逐出。The block must stay in the cache until committed: until then, the cached copy is the only record of the modification; it cannot be written to its place on disk until after commit; and other reads in the same transaction must see the modifications. `log_write` 会注意到在单个事务期间某个块被多次写入，并将该块分配到日志中的同一槽位。这种优化通常称为 _absorption_。例如，包含多个文件的 inode 的磁盘块在事务内被多次写入是很常见的。通过将多个磁盘写入吸收为一个，文件系统可以节省日志空间，并可以获得更好的性能，因为只需将磁盘块的一个副本写入磁盘。

#### log_write()
- 把 buffer 的磁盘块号保存到 log 中，同时做一个优化（absorbtion）
    - 找到一个同一磁盘块的拷贝时，用新的修改覆盖它即可（直接说使用原来留下的 slot）
        - 因为反正前一个修改会被覆盖
- 同时把 buffer 的 refcnt +1（避免被替换出去）

#### end_op()
- 首先将计数 log.outstanding -1（系统调用数 -1）
- 如果此时计数变成 0，则调用 commit() 进行 commit
There are four stages in this process. `write_log()` copies each block modified in the transaction from the buffer cache to its slot in the log on disk. `write_head()` writes the header block to disk: this is the commit point, and a crash after the write will result in recovery replaying the transaction's writes from the log,
![[Pasted image 20241227224518.png]]
- commit() 分为 4 个步骤
- write_log()：把具体的内容写入 log 对应的磁盘块里
    - 通过调用 bread() 读取磁盘块、memmove() 内存复制、bwrite() 写入磁盘完成
        - 需要写入的磁盘区域 \(\to\) 对应的 log 块
    - 代码如下，注意这里的 log 数组中记录的磁盘块写入对应**下标+1**的 log 块中，因为第一块留给了 log 的 head
- write_head()：把 log 块的头信息写入磁盘
    - log 块区域的第一块是 log 的 header 块
    - 这一个区域完成之后就实现了 commit() 的功能，就来来执行真正的写操作
- install_trans()：执行原来要求的写操作
    - 通过调用 bread() 读取磁盘块、memmove() 内存复制、bwrite() 写入磁盘完成
        - 对应的 log 块 \(\to\) 需要写入的磁盘区域
- write_head()：将磁盘块中的头信息清空（将计数归零）

#### recover_from_log()
`recover_from_log` is called from `initlog`, which is called from `fsinit` during boot before the first user process runs. 
![[Pasted image 20241227224745.png]]

An example use of the log occurs in `filewrite`. The transaction looks like this:
![[Pasted image 20250106144343.png]]
This code is wrapped in a loop that breaks up large writes into individual transactions of just a few sectors a time, to avoid overflowing the log. The call to `writei` writes many blocks as part of this transaction: the file's inode, one or more bitmap blocks, and some data blocks.
# 8.7 Code: Block allocator
The program `mkfs` sets the bits corresponding to the boot sector, superblock, log blocks, inode blocks, and bitmap blocks. 这个循环被分成两块。外层循环读位图的每一块。内层循环检查这一块中的所有 `BPB` 那么多个位。两个进程可能同时申请空闲块，这就有可能导致竞争，但事实上块缓冲只允许一个进程同时只使用一个块。

`bfree` finds the right block and clears the right bit. 同样，`bread` 和 `brelse` 的块的互斥使用使得无需再特意加锁。

# 8.8 Inode layer
_inode_ 有两个相关意思: It might refer to the on-disk structure containing a file's size and list of data block numbers. Or "inode" might refer to an in-memory inode, which contains a copy of the on-disk inode as well as extra information needed within the kernel.

The `type` field distinguishes between files, directories, and special files (devices). A type of zero indicates that an on-disk inode is free. The `nlink` field counts the number of directory entries that refer to this inode.

The kernel keeps the set of active inodes in memory in a table called `itable`. The kernel stores an inode in memory only if there are C pointers referring to that inode. The `ref` field counts the number of C pointers referring to the in-memory inode, and the kernel discards the inode from memory if the reference count drops to zero.

There are four lock or lock-like mechanisms in xv6's inode code. 
- `itable.lock` protects the invariant that an inode is present in the inode table at most once, and the invariant that an in-memory inode's `ref` field counts the number of in-memory pointers to the inode.
- Each in-memory inode has a `lock` field containing a sleep-lock, which ensures exclusive access to the inode's fields (such as file length) as well as to the inode's file or directory content blocks.
- Finally, each inode contains a `nlink` field that counts the number of directory entries that refer to a file; xv6 won't free an inode if its link count is greater than zero.

A `struct inode` pointer returned by `iget()` is guaranteed to be valid until the corresponding call to `iput()`; `iget()` provides non-exclusive access to an inode, so that there can be mant pointers to the same inode. 

The `struct inode` that `iget` returns may not have any useful content. In order to ensure it holds a copy of the on-disk inode, code must call `ilock`. This locks the inode (so that no other process can `ilock` it) and reads the inode from the disk, if it has not already been read. `iunlock` releases the lock on the inode.

Code that modifies an in-memory inode writes it to disk with `iupdate`.

# 8.9 Code: Inodes
To allocate a new inode, xv6 calls `ialloc`. 

Code must lock the inode using `ilock` before reading or writing its metadata or content. `ilock` uses a sleep-lock for this purpose. 

`iput` releases a C pointer to an inode by decrementing the reference count. If this is the last reference, the inode's slot in the inode table is now free and can be re-used for a different inode.

If `iput` sees that there are no C pointer references to an inode and that the inode has no links to it (occurs in no directory), then the inode and its data blocks must be freed. `iput` calls `itrunc` to  truncate the file to zero bytes, freeing the data blocks; sets the inode type to 0 (unallocated); and writes the inode to disk.
内存中的 _inode_ 比磁盘上的 _inode_ 多了几个属性，首先是设备号，Linux 里面一切皆文件，设备也是文件，所以设备号来表示什么设备。但是 xv6 没这么复杂，这里主要就是来区分磁盘的主盘和从盘。dev=1 时为从盘，dev=0 时为主盘，这个值在读写磁盘的时候用到，用它来设置磁盘的 device 寄存器来指定主盘从盘。

ref **表示引用数，这个要与 link 链接数作区别，目前可以暂且理解为 link 为磁盘上文件之间的关系，而 ref 主要用于内存中引用该文件的次数**，比如 close() 关闭文件使引用数减 1。这部分在文件系统调用的时候再作详细讲解。

valid **表示是否有效，跟磁盘那里缓存块中的数据是否有效一个意思，如果缓存中的数据是从磁盘中读取过来的，则有效。通常无效是因为 inode 刚分配，所以里面的数据无效**。

整个 inode 缓存区有一把自旋锁，每个 inode 缓存有把休眠锁，为什么如此，道理还是同磁盘和缓存块。首先它们都是公共资源，需要锁来避免竞争条件。再者 icache 的作用就是组织管理 inode，像是一个分配器，访问icache 的临界区的时间是很短的，使用自旋锁就行。而一个进程对某个 inode 的使用时间可能很长，最好使用休眠锁，其他进程也想要获取该 inode 的使用权时就休眠让出 CPU 提高性能。

在释放 inode 的情况下，iput 中的锁定协议值得仔细研究。一个危险是并发线程可能正在 ilock 中等待使用此 inode（例如，读取文件或列出目录），并且不会准备好发现 inode 不再被分配。这种情况不可能发生，因为如果系统调用没有指向内存 inode 的链接并且 ip->ref 为 1，则系统调用无法获取指向内存 inode 的指针。该引用是调用 iput 的线程拥有的引用。确实，iput 在其 itable.lock 关键部分之外检查引用计数是否为 1，但此时已知链接计数为零，因此没有线程会尝试获取新的引用。另一个主要危险是并发调用 ialloc 可能会选择 iput 正在释放的相同 inode。这只有在 iupdate 写入磁盘以使 inode 具有类型零之后才会发生。这种竞争是良性的；分配线程会礼貌地等待获取 inode 的睡眠锁，然后再读取或写入 inode，此时 iput 就完成了。

There is a challenging interaction between `iput()` and crashes. `iput()` doesn't truncate a file immediately when the link count for the file drops to zero, because some process might still hold a reference to the inode in memory: a process might still be reading and writing to the file, because it successfully opened it. 如果一个文件的最后一个进程在崩溃时尚未关闭文件，文件的 inode 和数据块可能会变成“悬挂”的状态。它们仍然占用磁盘空间，但无法通过目录访问到它。为了避免这种情况，文件系统需要有一种机制来处理崩溃恢复，确保这种“悬挂”文件能够在系统恢复后被清理，或者确保它不会导致磁盘空间泄漏。

## 分配 inode
分配空闲 dinode 的方法就是从头至尾查找空闲 dinode
![[Pasted image 20250101125133.png]]
**分配数据块的时候有位图来组织管理，所以分配数据块的时候就“从头至尾”的查询空闲位，**，而 dinode 没有组织管理的机制，所以就直接从头至尾的查询 dinode 的使用情况。

分配了该 dinode 需要在磁盘上也将其标记为已分配，因为目前是在内存中操作的，得同步到磁盘上去，所以直接调用 `log_write()` 将该 dinode 所在的缓存块同步到磁盘。当然并未真正地直接写到磁盘，只是将该缓存数据标记为脏。

磁盘上的 dinode 已分配，得到了 inode 号，但是文件系统实际工作的时候使用的是内存中的 inode 缓存，所以调用 `iget()` 来分配（获取）一个内存中的 inode 来缓存 dinode 数据：

如果磁盘上的 dinode 在 icache 中已有缓存，那么直接将该 inode 的引用数加 1，再返回 inode 就行。如果没有缓存则分配一个空闲的 inode，根据参数初始化 inode。

## 使用修改 inode
使用 inodeinode 之前需要加锁：
![[Pasted image 20250101140717.png]]

分配 inode 的时候并未从磁盘中 dinode 读入数据，只是将 inode 的 valid 置 0 表示数据无效，正式读入 dinode 数据在这加锁的时候进行。

对缓存中 inode 的修改需要同步到磁盘上的 dinode：
![[Pasted image 20250101140957.png]]

## 索引
inode 的索引部分用来指向数据块
![[Pasted image 20250101141206.png]]

返回索引 bn 指向的数据块块号，如果该数据块未分配，则分配之后再返回该块号。
![[Pasted image 20250101141353.png]]

截断 inode，将 inode 所指向的数据块全部释放，相当于删除文件

# 8.10 Code: Inode content
The on-disk inode structure, `struct dinode`, contains a size and an array of block numbers.
![[Pasted image 20250101212307.png]]
Thus the first 12kB (`NDIRECT X BSIZE`) bytes of a file can be loaded from blocks listed in the inode, while the next 256kB (`NINDIRECT X BSIZE`) bytes can only be loaded after consulting the indirect block. 

# 目录
目录项的主要作用就是将 inode 和文件名联系起来
![[Pasted image 20250101215607.png]]
**因此根据文件名查找文件的是指就是在目录文件中查找目录项的过程，具体的查找方式就是一个个的比对目录项的名称和要查找的文件名是否相同，如果相同，则找到，反之说明该目录下并没有要查找的文件**。

## 添加目录项

# 路径
**另外像 /a/b/c/a/b/c 这种路径以 `'/'` 开头表示绝对路径，a/b/ca/b/c 这种不以 `'/'` 开头的表示相对路径**。

不论哪一种路径表示，都需要一个路径解析函数，将其中一个个文件名给提取出来


# Real world
xv6中的buffer cache采用了一个非常简单的链表来对LRU进行剔除，但是实际的操作系统中采用了hash表和heap来进行LRU剔除，且加入了虚拟内存系统来支持memory-mapped文件。

Xv6's logging system is inefficient. A commit cannot occur concurrently with file-system system calls. The system logs entire blocks, even if only a few bytes in a block are changed. It performs synchronous log writes, a block at a time, each of which is likely to require an entire disk rotation time.

Logging is not the only way to provide crash recovery. Early file systems used a scavenger during reboot to examine every file and directory and the block and inode free lists, looking for and resolving inconsistencies
在目录树中采用了线性扫描disk block的方式进行查找，在disk block较多的情况下会消耗很多时间，因此Windows的NTFS等文件系统将文件夹用balanced tree进行表示，以确保对数事件的查找。

xv6要求文件系统只能有一个硬盘设备，但是现代操作系统可以采用RAID或者软件的方式来将很多硬盘组成一个逻辑盘

现代操作系统还应该具备的其他特性：snapshots、增量式备份。