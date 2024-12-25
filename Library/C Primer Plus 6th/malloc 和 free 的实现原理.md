![[Pasted image 20241213205123.png]]
**内存映射段(mmap)**的作用是：内核将硬盘文件的内容直接映射到内存，任何应用程序都可通过 Linux 的 mmap() 系统调用请求这种映射。
- 内存映射是一种方便高效的文件 I/O 方式， 因而被用于装载动态共享库。
- 用户也可创建匿名内存映射，该映射没有对应的文件，可用于存放程序数据
- mmap 映射区向下扩展，堆向上扩展，两者相对扩展，直到耗尽虚拟地址空间中的剩余区域。

在 Linux 中，若通过 malloc() 请求一大块内存，C 运行库将创建一个匿名内存映射，而不使用堆内存。“大块”意味着比阈值MMAP_THRESHOLD还大，缺省为 128KB，可通过 mallopt() 调整。

**实际中系统分配较大的堆空间，进程通过malloc()库函数在堆上进行空间动态分配，堆如果不够用malloc可以进行系统调用，增大brk的值。**

# 相关系统调用
- brk()和sbrk()

由之前的进程地址空间结构分析可以知道，要增加一个进程实际的可用堆大小，就需要将break指针向高地址移动。Linux通过**brk和sbrk系统调用**操作break指针。两个系统调用的原型如下：
```c
#include <unistd.h>
int brk(void *addr);
void *sbrk(intptr_t increment);
```
brk函数将break指针直接设置为某个地址，而sbrk将break指针从当前位置移动increment所指定的增量。brk在执行成功时返回0，否则返回-1并设置errno为ENOMEM；sbrk成功时返回break指针移动之前所指向的地址，否则返回(void *)-1。

- mmap函数
![[Pasted image 20241213205526.png]]
mmap函数第一种用法是映射磁盘文件到内存中；而malloc使用的mmap函数的第二种用法，即匿名映射，匿名映射不映射磁盘文件，而是向映射区申请一块内存。  
munmap函数是用于释放内存，第一个参数为内存首地址，第二个参数为内存的长度。接下来看下mmap函数的参数

当申请小内存的时，malloc使用sbrk分配内存；当申请大内存时，使用mmap函数申请内存；但是这只是分配了虚拟内存，还没有映射到物理内存，当访问申请的内存时，才会因为缺页异常，内核分配物理内存。
- 分配内存 < DEFAULT_MMAP_THRESHOLD，走__brk，从内存池获取，失败的话走brk系统调用
- 分配内存 > DEFAULT_MMAP_THRESHOLD，走__mmap，直接调用mmap系统调用

# malloc 实现方案
**malloc采用的是内存池的实现方式**，malloc内存池实现方式更类似于STL分配器和memcached的内存池，先申请一大块内存，然后将内存分成不同大小的内存块，然后用户申请内存时，直接从内存池中选择一块相近的内存块即可。
![[Pasted image 20241213205654.png]]
内存池保存在bins这个长128的数组中，每个元素都是一双向个链表。
- bins[0]目前没有使用
- bins[1]的链表称为unsorted_list，用于维护free释放的chunk。
- bins\[2,63)的区间称为small_bins，用于维护＜512字节的内存块，其中每个元素对应的链表中的chunk大小相同，均为index\*8。
- bins\[64,127)称为large_bins，用于维护>512字节的内存块，每个元素对应的链表中的chunk大小不同，index越大，链表中chunk的内存大小相差越大，例如: 下标为64的chunk大小介于\[512, 512+64)，下标为95的chunk大小介于\[2k+1,2k+512)。同一条链表上的chunk，按照从小到大的顺序排列。

malloc将内存分成了大小不同的chunk，然后通过bins来组织起来。malloc将相似大小的chunk（图中可以看出同一链表上的chunk大小差不多）用双向链表链接起来，这样一个链表被称为一个bin。malloc一共维护了128个bin，并使用一个数组来存储这些bin。数组中第一个为**unsorted bin**，数组编号前2到前64的bin为**small bins**，同一个small bin中的chunk具有相同的大小，两个相邻的small bin中的chunk大小相差8bytes。small bins后面的bin被称作**large bins**。large bins中的每一个bin分别包含了一个给定范围内的chunk，其中的chunk按大小序排列。large bin的每个bin相差64字节。

malloc除了有unsorted bin，small bin，large bin三个bin之外，还有一个**fast bin**。一般的情况是，程序在运行时会经常需要申请和释放一些较小的内存空间。当分配器合并了相邻的几个小的 chunk 之后，也许马上就会有另一个小块内存的请求，这样分配器又需要从大的空闲内存中切分出一块，这样无疑是比较低效的，故而，malloc 中在分配过程中引入了 fast bins，不大于 max_fast(默认值为 64B)的 chunk 被释放后，首先会被放到 fast bins中，fast bins 中的 chunk 并不改变它的使用标志 P。这样也就无法将它们合并，当需要给用户分配的 chunk 小于或等于 max_fast 时，malloc 首先会在 fast bins 中查找相应的空闲块，然后才会去查找 bins 中的空闲 chunk。在某个特定的时候，malloc 会遍历 fast bins 中的 chunk，将相邻的空闲 chunk 进行合并，并将合并后的 chunk 加入 unsorted bin 中，然后再将 unsorted bin 里的 chunk 加入 bins 中。

unsorted bin 的队列使用 bins 数组的第一个，如果被用户释放的 chunk 大于 max_fast，或者 fast bins 中的空闲 chunk 合并后，这些 chunk 首先会被放到 unsorted bin 队列中，在进行 malloc 操作的时候，如果在 fast bins 中没有找到合适的 chunk，则malloc 会先在 unsorted bin 中查找合适的空闲 chunk，然后才查找 bins。如果 unsorted bin 不能满足分配要求。 malloc便会将 unsorted bin 中的 chunk 加入 bins 中。然后再从 bins 中继续进行查找和分配过程。从这个过程可以看出来，**unsorted bin 可以看做是 bins 的一个缓冲区，增加它只是为了加快分配的速度。**

除了上述四种bins之外，malloc还有三种内存区。

- 当fast bin和bins都不能满足内存需求时，malloc会设法在**top chunk**中分配一块内存给用户；top chunk为在mmap区域分配一块较大的空闲内存模拟sub-heap。（比较大的时候） >top chunk是堆顶的chunk，堆顶指针brk位于top chunk的顶部。移动brk指针，即可扩充top chunk的大小。当top chunk大小超过128k(可配置)时，会触发malloc_trim操作，调用sbrk(-size)将内存归还操作系统。
    
- 当chunk足够大，fast bin和bins都不能满足要求，甚至top chunk都不能满足时，malloc会从mmap来直接使用内存映射来将页映射到进程空间，这样的chunk释放时，直接解除映射，归还给操作系统。（极限大的时候）
    
- Last remainder是另外一种特殊的chunk，就像top chunk和mmaped chunk一样，不会在任何bins中找到这种chunk。当需要分配一个small chunk,但在small bins中找不到合适的chunk，如果last remainder chunk的大小大于所需要的small chunk大小，last remainder chunk被分裂成两个chunk，其中一个chunk返回给用户，另一个chunk变成新的last remainder chunk。（这个应该是fast bins中也找不到合适的时候，用于极限小的）
![[Pasted image 20241213210442.png]]
由之前的分析可知malloc利用chunk结构来管理内存块，malloc就是由不同大小的chunk链表组成的。malloc会给用户分配的空间的前后加上一些控制信息，用这样的方法来记录分配的信息，以便完成分配和释放工作。chunk指针指向chunk开始的地方,图中的mem指针才是真正返回给用户的内存指针。
- chunk 的第二个域的最低一位为**P**，它**表示前一个块是否在使用中**，P 为 0 则表示前一个 chunk 为空闲，这时chunk的第一个域 prev_size 才有效，prev_size 表示前一个 chunk 的 size，程序可以使用这个值来找到前一个 chunk 的开始地址。当 P 为 1 时，表示前一个 chunk 正在使用中，prev_size程序也就不可以得到前一个 chunk 的大小。不能对前一个 chunk 进行任何操作。**malloc分配的第一个块总是将 P 设为 1，以防止程序引用到不存在的区域。**
- Chunk 的第二个域的倒数第二个位为**M**，他表示当前 chunk 是**从哪个内存区域获得的虚拟内存**。M 为 1 表示该 chunk 是从 mmap 映射区域分配的，否则是从 heap 区域分配的。
- Chunk 的第二个域倒数第三个位为 **A**，表示该 chunk **属于主分配区或者非主分配区**，如果属于非主分配区，将该位置为 1，否则置为 0。
当chunk空闲时，其M状态是不存在的，只有AP状态，原本是用户数据区的地方存储了四个指针，指针fd指向后一个空闲的chunk,而bk指向前一个空闲的chunk，malloc通过这两个指针将大小相近的chunk连成一个双向链表。在large bin中的空闲chunk，还有两个指针，fd_nextsize和bk_nextsize，用于加快在large bin中查找最近匹配的空闲chunk。不同的chunk链表又是通过bins或者fastbins来组织的。

# malloc 内存分配流程
1. 如果分配内存<512字节，则通过内存大小定位到smallbins对应的index上(floor(size/8))
    
    - 如果smallbins[index]为空，进入步骤3
    - 如果smallbins[index]非空，直接返回第一个chunk
2. 如果分配内存>512字节，则定位到largebins对应的index上
    
    - 如果largebins[index]为空，进入步骤3
    - 如果largebins[index]非空，扫描链表，找到第一个大小最合适的chunk，如size=12.5K，则使用chunk B，剩下的0.5k放入unsorted_list中
3. 遍历unsorted_list，查找合适size的chunk，如果找到则返回；否则，将这些chunk都归类放到smallbins和largebins里面
    
4. index++从更大的链表中查找，直到找到合适大小的chunk为止，找到后将chunk拆分，并将剩余的加入到unsorted_list中
    
5. 如果还没有找到，那么使用top chunk
    
6. 或者，内存<128k，使用brk；内存>128k，使用mmap获取新内存
此外，调用free函数时，它将用户释放的内存块连接到空闲链上。到最后，空闲链会被切成很多的小内存片段，如果这时用户申请一个大的内存片段，那么空闲链上可能没有可以满足用户要求的片段了。于是，malloc函数请求延时，并开始在空闲链上翻箱倒柜地检查各内存片段，对它们进行整理，将相邻的小空闲块合并成较大的内存块。

- 虚拟内存并不是每次malloc后都增长，是与上一节说的堆顶没发生变化有关，因为可重用堆顶内剩余的空间，这样的malloc是很轻量快速的。
- 如果虚拟内存发生变化，基本与分配内存量相当，因为虚拟内存是计算虚拟地址空间总大小。
- 物理内存的增量很少，是因为malloc分配的内存并不就马上分配实际存储空间，只有第一次使用，如第一次memset后才会分配。
- 由于每个物理内存页面大小是4k，不管memset其中的1k还是5k、7k，实际占用物理内存总是4k的倍数。所以物理内存的增量总是4k的倍数。  
    **因此，不是malloc后就马上占用实际内存，而是第一次使用时发现虚存对应的物理页面未分配，产生缺页中断，才真正分配物理页面，同时更新进程页面的映射关系。这也是Linux虚拟内存管理的核心概念之一。**

# 内存碎片
free释放内存时，有两种情况：

1. chunk和top chunk相邻，则和top chunk合并
2. chunk和top chunk不相邻，则直接插入到unsorted_list中

如上图示: top chunk是堆顶的chunk，堆顶指针brk位于top chunk的顶部。移动brk指针，即可扩充top chunk的大小。**当top chunk大小超过128k(可配置)时，会触发malloc_trim操作，调用sbrk(-size)将内存归还操作系统。**


# glibc 源码分析

https://mp.weixin.qq.com/s/pdv5MMUQ9ACpeCpyGnxb1Q

glibc 下的内存管理器（ptmalloc），是一个内存池，应用程序的申请内存或者释放内存，都是在该内存池中实现，只有满足 ptmalloc 的某些特定条件之后，ptmalloc 才会调用 sys_trim 函数，将内存归还OS

下面是针对glibc中内存管理主要函数**malloc**和**free**的实现原理

1、下面是application申请内存时候的宏观图
![[Pasted image 20241213212718.png]]

2、glibc的分配和释放远比我想象复杂的多，里面涉及到bin概念
fast bins，small bins，largebins，top chunk，mmaped chunk以及lastremainder chunk
![[Pasted image 20241213212749.png]]这里要提到一个很重要的概念，内存的延迟分配，只有在真正访问一个地址的时候才建立这个地址的物理映射，这是 Linux 内存管理的基本思想之一。Linux 内核在用户申请内存的时候，只是给它分配了一个线性区（也就是虚拟内存），并没有分配实际物理内存；只有当用户使用这块内存的时候，内核才会分配具体的物理页面给用户，这时候才占用宝贵的物理内存。内核释放物理页面是通过释放线性区，找到其所对应的物理页面，将其全部释放的过程。

进程的内存结构，在内核中，是用mm_struct来表示的，其定义如下：
![[Pasted image 20241213212941.png]]
在上述mm_struct结构中：

- [start_code,end_code)表示代码段的地址空间范围。
    
- [start_data,end_start)表示数据段的地址空间范围。
    
- [start_brk,brk)分别表示heap段的起始空间和当前的heap指针。
    
- [start_stack,end_stack)表示stack段的地址空间范围。
    
- mmap_base表示memory mapping段的起始地址。

C语言的动态内存分配基本函数是 malloc()，在 Linux 上的实现是通过内核的 brk 系统调用。brk()是一个非常简单的系统调用， 只是简单地改变mm_struct结构的成员变量 brk 的值。
![[Pasted image 20241213213012.png]]
映射关系分为以下两种：
- 文件映射: 磁盘文件映射进程的虚拟地址空间，使用文件内容初始化物理内存。
    
- 匿名映射: 初始化全为0的内存空间

映射关系是否共享，可以分为:
- 私有映射(MAP_PRIVATE)：多进程间数据共享，修改不反应到磁盘实际文件，是一个copy-on-write（写时复制）的映射方式。
- 共享映射(MAP_SHARED):多进程间数据共享，修改反应到磁盘实际文件中。


因此，整个映射关系总结起来分为以下四种:
- 私有文件映射 多个进程使用同样的物理内存页进行初始化，但是各个进程对内存文件的修改不会共享，也不会反应到物理文件中
- 私有匿名映射: mmap会创建一个新的映射，各个进程不共享，这种使用主要用于分配内存(malloc分配大内存会调用mmap)。例如开辟新进程时，会为每个进程分配虚拟的地址空间，这些虚拟地址映射的物理内存空间各个进程间读的时候共享，写的时候会copy-on-write。
- 共享文件映射:- 多个进程通过虚拟内存技术共享同样的物理内存空间，对内存文件的修改会反应到实际物理文件中，也是进程间通信(IPC)的一种机制。
- 共享匿名映射:  这种机制在进行fork的时候不会采用写时复制，父子进程完全共享同样的物理内存页，这也就实现了父子进程通信(IPC)。

在 Linux 中，内存映射文件时，基于映射的特性和映射区域的共享性，可以进一步细分为四种类型的映射：**私有文件映射**、**私有匿名映射**、**共享文件映射** 和 **共享匿名映射**。这四种映射关系分别有不同的行为和适用场景。

### 1. **私有文件映射（Private File Mapping）**

- **描述**：私有文件映射是将文件映射到进程的虚拟内存区域，但该映射区域对其他进程不可见。如果进程对该内存区域进行了修改，这些修改不会反映到原文件中。私有文件映射通常与写时复制（Copy-on-Write，COW）机制一起工作：当进程尝试修改映射区域时，操作系统不会直接修改文件，而是创建一个新的内存页来容纳修改的数据。
- **行为**：
    - 映射区域仅对当前进程可见。
    - 修改映射区域不会影响原文件内容。
    - 支持写时复制机制。
- **典型用法**：用于临时访问文件内容，但不想修改文件本身的场景。可以用于实现内存缓存。

#### 系统调用示例：

```c
char *addr = mmap(NULL, length, PROT_READ | PROT_WRITE, MAP_PRIVATE, fd, 0);
```

- `MAP_PRIVATE`：表示私有映射，修改映射内容不会影响原文件。

### 2. **私有匿名映射（Private Anonymous Mapping）**

- **描述**：私有匿名映射是指没有文件或设备与该内存区域关联，映射区域仅供进程使用，并且修改这些内存内容不会影响任何文件。它通常用于分配进程的私有内存区域，如堆、栈等。这种映射区域不会被写回任何文件，也不会共享给其他进程。
- **行为**：
    - 映射区域没有与任何文件关联。
    - 修改映射区域不会影响任何文件或外部数据。
    - 支持写时复制机制。
- **典型用法**：用于分配进程私有内存空间，通常用于堆、栈或临时数据存储。

#### 系统调用示例：

```c
char *addr = mmap(NULL, length, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
```

- `MAP_PRIVATE`：表示私有映射，修改映射内容不会影响原文件。
- `MAP_ANONYMOUS`：表示该映射没有与任何文件关联。

### 3. **共享文件映射（Shared File Mapping）**

- **描述**：共享文件映射是将文件映射到多个进程的虚拟内存中，多个进程可以共享该映射区域。任何进程在映射区域中的修改都会立即反映到其他进程的映射区域。这种映射适用于多个进程需要读取和修改相同的文件内容。
- **行为**：
    - 映射区域在多个进程之间共享。
    - 进程对映射区域的修改会立即反映到其他进程。
    - 不支持写时复制，修改会直接写回文件。
- **典型用法**：用于进程间共享数据（例如，多个进程读取和写入共享文件）。

#### 系统调用示例：

```c
char *addr = mmap(NULL, length, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
```

- `MAP_SHARED`：表示共享映射，修改映射内容会影响原文件。

### 4. **共享匿名映射（Shared Anonymous Mapping）**

- **描述**：共享匿名映射是指没有文件与内存区域关联，但该内存区域在多个进程之间共享。这种映射通常用于进程间共享数据，通常通过共享内存实现。这些内存区域的内容不会映射回任何文件，因此通常只在进程间交换数据时使用。
- **行为**：
    - 映射区域没有与任何文件关联。
    - 映射区域在多个进程之间共享。
    - 修改内容会影响所有共享该内存区域的进程。
- **典型用法**：进程间通信（IPC）中，多个进程需要共享数据而不依赖于磁盘上的文件。

#### 系统调用示例：

```c
char *addr = mmap(NULL, length, PROT_READ | PROT_WRITE, MAP_SHARED | MAP_ANONYMOUS, -1, 0);
```

- `MAP_SHARED`：表示共享映射，修改映射内容会影响所有共享此区域的进程。
- `MAP_ANONYMOUS`：表示没有与文件关联。

---

### 这四种映射的总结

|映射类型|是否与文件关联|是否共享|修改影响|写时复制（COW）|典型用途|
|---|---|---|---|---|---|
|**私有文件映射**|与文件关联|否|不影响文件|支持|临时修改文件内容但不影响文件本身|
|**私有匿名映射**|无文件关联|否|不影响文件|支持|分配进程的私有内存区域（如堆、栈）|
|**共享文件映射**|与文件关联|是|影响文件|不支持|进程间共享文件内容|
|**共享匿名映射**|无文件关联|是|影响所有进程|不支持|进程间共享内存（如共享内存）|

### 主要应用场景：

- **私有文件映射**：用于临时操作文件数据，例如缓存文件内容，但不修改文件本身。
- **私有匿名映射**：用于进程内部的私有数据存储，例如堆、栈或临时内存。
- **共享文件映射**：用于多个进程共享文件内容，适合进程间读取/写入相同文件。
- **共享匿名映射**：用于进程间共享内存，通常用于进程间通信（IPC）和共享数据。

这四种映射方式提供了灵活的内存管理机制，使得进程能够高效、灵活地访问和共享内存或文件内容。根据具体应用需求，选择合适的映射方式可以提高程序性能、减少内存使用并简化进程间通信。

# 内存管理
1 ptmalloc：ptmalloc是隶属于glibc(GNU Libc)的一款内存分配器，现在在Linux环境上，我们使用的运行库的内存分配(malloc/new)和释放(free/delete)就是由其提供。

2 BSD Malloc：BSD Malloc 是随 4.2 BSD 发行的实现，包含在 FreeBSD 之中，这个分配程序可以从预先确实大小的对象构成的池中分配对象。它有一些用于对象大小的size 类，这些对象的大小为 2 的若干次幂减去某一常数。所以，如果您请求给定大小的一个对象，它就简单地分配一个与之匹配的 size 类。这样就提供了一个快速的实现，但是可能会浪费内存。

3 Hoard：编写 Hoard 的目标是使内存分配在多线程环境中进行得非常快。因此，它的构造以锁的使用为中心，从而使所有进程不必等待分配内存。它可以显著地加快那些进行很多分配和回收的多线程进程的速度。

4 TCMalloc：Google 开发的内存分配器，在不少项目中都有使用，例如在 Golang 中就使用了类似的算法进行内存分配。它具有现代化内存分配器的基本特征：对抗内存碎片、在多核处理器能够 scale。据称，它的内存分配速度是 glibc2.3 中实现的 malloc的数倍。

## 5 ptmalloc
![[Pasted image 20241213214150.png]]
因为本次事故就是用的运行库函数new/delete进行的内存分配和释放，所以本文将着重分析glibc下的内存分配库ptmalloc。

在c/c++中，我们分配内存是在堆上进行分配，那么这个堆，在glibc中是怎么表示的呢？

我们先看下堆的结构声明:
![[Pasted image 20241213214219.png]]
在堆的上述定义中，ar_ptr是指向分配区的指针，堆之间是以链表方式进行连接，

### 分配区 arena
ptmalloc对进程内存是通过一个个Arena来进行管理的。

在ptmalloc中，分配区分为主分配区(arena)和非主分配区(narena)，分配区用struct malloc_state来表示。主分配区和非主分配区的区别是 主分配区可以使用sbrk和mmap向os申请内存，而非分配区只能通过mmap向os申请内存。

当一个线程调用malloc申请内存时，该线程先查看线程私有变量中是否已经存在一个分配区。如果存在，则对该分配区加锁，加锁成功的话就用该分配区进行内存分配；失败的话则搜索环形链表找一个未加锁的分配区。如果所有分配区都已经加锁，那么malloc会开辟一个新的分配区加入环形链表并加锁，用它来分配内存。释放操作同样需要获得锁才能进行。

需要注意的是，非主分配区是通过mmap向os申请内存，一次申请64MB，一旦申请了，该分配区就不会被释放，为了避免资源浪费，ptmalloc对分配区是有个数限制的

对于32位系统，分配区最大个数 = 2 * CPU核数 + 1

对于64位系统，分配区最大个数 = 8 * CPU核数 + 1

堆管理结构：
![[Pasted image 20241213214454.png]]
![[Pasted image 20241213214502.png]]

每一个进程只有一个主分配区和若干个非主分配区。主分配区由main线程或者第一个线程来创建持有。主分配区和非主分配区用环形链表连接起来。分配区内有一个变量mutex以支持多线程访问。
![[Pasted image 20241213214551.png]]
在前面有提到，在每个分配区中都有一个变量mutex来支持多线程访问。每个线程一定对应一个分配区，但是一个分配区可以给多个线程使用，同时一个分配区可以由一个或者多个的堆组成，同一个分配区下的堆以链表方式进行连接，它们之间的关系如下图：
![[Pasted image 20241213214621.png]]
一个进程的动态内存，由分配区管理，一个进程内有多个分配区，一个分配区有多个堆，这就组成了复杂的进程内存管理结构。
![[Pasted image 20241213214702.png]]
需要注意几个点：

- 主分配区通过brk进行分配，非主分配区通过mmap进行分配
    
- 非主分配区虽然是mmap分配，但是和大于128K直接使用mmap分配没有任何联系。大于128K的内存使用mmap分配，使用完之后直接用ummap还给系统
    
- 每个线程在malloc会先获取一个area，使用area内存池分配自己的内存，这里存在竞争问题
    
- 为了避免竞争，我们可以使用线程局部存储，thread cache（tcmalloc中的tc正是此意），线程局部存储对area的改进原理如下：
    
- 如果需要在一个线程内部的各个函数调用都能访问、但其它线程不能访问的变量（被称为static memory local to a thread 线程局部静态变量），就需要新的机制来实现。这就是TLS。
    
- thread cache本质上是在static区为每一个thread开辟一个独有的空间，因为独有，不再有竞争
    
- 每次malloc时，先去线程局部存储空间中找area，用thread cache中的area分配存在thread area中的chunk。当不够时才去找堆区的area。

### chunk
ptmalloc通过malloc_chunk来管理内存，给User data前存储了一些信息，使用边界标记区分各个chunk。

chunk定义如下:
![[Pasted image 20241213214755.png]]
- prev_size: 如果前一个chunk是空闲的，则该域表示前一个chunk的大小，如果前一个chunk不空闲，该域无意义。
一段连续的内存被分成多个chunk，prev_size记录的就是相邻的前一个chunk的size，知道当前chunk的地址，减去prev_size便是前一个chunk的地址。prev_size主要用于相邻空闲chunk的合并。

- size ：当前 chunk 的大小，并且记录了当前 chunk 和前一个 chunk 的一些属性，包括前一个 chunk 是否在使用中，当前 chunk 是否是通过 mmap 获得的内存，当前 chunk 是否属于非主分配区。
    
- fd 和 bk ：指针 fd 和 bk 只有当该 chunk 块空闲时才存在，其作用是用于将对应的空闲 chunk 块加入到空闲chunk 块链表中统一管理，如果该 chunk 块被分配给应用程序使用，那么这两个指针也就没有用（该 chunk 块已经从空闲链中拆出）了，所以也当作应用程序的使用空间，而不至于浪费。
    
- fd_nextsize 和 bk_nextsize: 当前的 chunk 存在于 large bins 中时， large bins 中的空闲 chunk 是按照大小排序的，但同一个大小的 chunk 可能有多个，增加了这两个字段可以加快遍历空闲 chunk ，并查找满足需要的空闲 chunk ， fd_nextsize 指向下一个比当前 chunk 大小大的第一个空闲 chunk ， bk_nextszie 指向前一个比当前 chunk 大小小的第一个空闲 chunk 。（同一大小的chunk可能有多块，在总体大小有序的情况下，要想找到下一个比自己大或小的chunk，需要遍历所有相同的chunk，所以才有fd_nextsize和bk_nextsize这种设计） 如果该 chunk 块被分配给应用程序使用，那么这两个指针也就没有用（该chunk 块已经从 size 链中拆出）了，所以也当作应用程序的使用空间，而不至于浪费。

正如上面所描述，在ptmalloc中，为了尽可能的节省内存，使用中的chunk和未使用的chunk在结构上是不一样的。
![[Pasted image 20241213214845.png]]

在上图中：

- chunk指针指向chunk开始的地址
    
- mem指针指向用户内存块开始的地址。
    
- p=0时，表示前一个chunk为空闲，prev_size才有效
    
- p=1时，表示前一个chunk正在使用，prev_size无效 p主要用于内存块的合并操作；ptmalloc 分配的第一个块总是将p设为1, 以防止程序引用到不存在的区域
    
- M=1 为mmap映射区域分配；M=0为heap区域分配
    
- A=0 为主分配区分配；A=1 为非主分配区分配。
    

与非空闲chunk相比，空闲chunk在用户区域多了四个指针，分别为fd,bk,fd_nextsize,bk_nextsize，这几个指针的含义在上面已经有解释，在此不再赘述。

空闲chunk
![[Pasted image 20241213214902.png]]