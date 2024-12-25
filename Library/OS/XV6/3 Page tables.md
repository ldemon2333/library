页表是最流行的机制，通过该机制，操作系统为每个进程提供自己的私有地址空间和内存。页表确定内存地址的含义以及可以访问物理内存的哪些部分。它们允许 xv6 隔离不同进程的地址空间并将它们多路复用到单个物理内存上。页表是一种流行的设计，因为它们提供了一个间接级别，允许操作系统执行许多技巧。xv6 执行了一些技巧：在多个地址空间中映射相同的内存（a trampoline page），并使用未映射的页面保护内核和用户堆栈。本章的其余部分将解释 RISC-V 硬件提供的页表以及 xv6 如何使用它们。

# 3.1 Paging hardware
xv6运行于Sv39 RISC-V，即在64位地址中只有最下面的39位被使用作为虚拟地址，其中底12位是页内偏移，高27位是页表索引，即4096字节作为一个page，一个进程的虚拟内存可以有 $2^{27}$ 个page，对应到页表中就是 $2^{27}$ 个page table entry (PTE)。每个PTE有一个44位的physical page number (PPN)用来映射到物理地址上和10位flag，总共需要54位，也就是一个PTE需要8字节存储。即每个物理地址的高44位是页表中存储的PPN，低12位是页内偏移，一个物理地址总共由56位构成。
![[Pasted image 20241212134431.png]]
在实际中，页表并不是作为一个包含了$2^{27}$个PTE的大列表存储在物理内存中的，而是采用了三级树状的形式进行存储，这样可以让页表分散存储。每个页表就是一页。第一级页表是一个4096字节的页，包含了512个PTE（因为每个PTE需要8字节），每个PTE存储了下级页表的页物理地址，第二级列表由512个页构成，第三级列表由512 * 512个页构成。因为每个进程虚拟地址的高27位用来确定PTE，对应到3级页表就是最高的9位确定一级页表PTE的位置，中间9位确定二级页表PTE的位置，最低9位确定三级页表PTE的位置。如下图所示。第一级根页表的物理页地址存储在`satp`寄存器中，每个CPU拥有自己独立的`satp`.
![[Pasted image 20241212134519.png]]
PTE flag可以告诉硬件这些相应的虚拟地址怎样被使用，比如`PTE_V`表明这个PTE是否存在，`PTE_R`、`PTE_W`、`PTE_X`控制这个页是否允许被读取、写入和执行，`PTE_U`控制user 
mode是否有权访问这个页，如果`PTE_U`=0，则只有supervisor mode有权访问这个页。

If any of three PTEs required to translate an address is not present, the paging hardware raises a *page-fault exception*, leaving it up to the kernel to handle the exception.

与图 3.1 中的单级设计相比，图 3.2 中的三级结构允许以内存高效的方式记录 PTE。在虚拟地址的大范围没有映射的常见情况下，三级结构可以省略整个页面目录。例如，如果应用程序仅使用从地址零开始的几个页面，则顶级页面目录的条目 1 到 511 无效，并且内核不必为 511 个中间页面目录分配页面。此外，内核也不必为这 511 个中间页面目录分配底层页面目录的页面。因此，在此示例中，三级设计为中间页面目录节省了 511 个页面，为底层页面目录节省了 511 × 512 个页面。

尽管 CPU 在执行加载或存储指令时会在硬件中遍历三级结构，但三级结构的一个潜在缺点是 CPU 必须从内存中加载三个 PTE，才能将加载/存储指令中的虚拟地址转换为物理地址。
为了避免从物理内存加载 PTE 的成本，RISC-V CPU 会将页表条目缓存在转换后备缓冲区 (TLB) 中。

每个 PTE 都包含标志位，用于告诉分页硬件允许如何使用相关的虚拟地址。PTE_V 表示 PTE 是否存在：如果未设置，则对页面的引用会导致异常（即不允许）。PTE_R 控制是否允许指令读取页面。PTE_W 控制是否允许指令写入页面。PTE_X 控制 CPU 是否可以将页面内容解释为指令并执行它们。PTE_U 控制是否允许用户模式下的指令访问页面；如果未设置 PTE_U，则只能在管理员模式下使用 PTE。图 3.2 显示了它的工作原理。标志和所有其他与页面硬件相关的结构均在 (kernel/riscv.h) 中定义

要告诉硬件使用页表，内核必须将根页表页的物理地址写入 satp 寄存器。每个 CPU 都有自己的 satp。CPU 将使用其自己的 satp 指向的页表来转换后续指令生成的所有地址。每个 CPU 都有自己的 satp，因此不同的 CPU 可以运行不同的进程，每个进程都有一个由其自己的页表描述的私有地址空间。

通常，内核会将所有物理内存映射到其页表中，以便它可以使用加载/存储指令读取和写入物理内存中的任何位置。由于页面目录位于物理内存中，因此内核可以通过使用标准存储指令写入 PTE 的虚拟地址来对页面目录中的 PTE 内容进行编程。

关于术语的一些说明。物理内存是指 DRAM 中的存储单元。一个字节的物理内存有一个地址，称为物理地址。指令仅使用虚拟地址，分页硬件将其转换为物理地址，然后发送到 DRAM 硬件以读取或写入存储。与物理内存和虚拟地址不同，虚拟内存不是物理对象，而是指内核提供的用于管理物理内存和虚拟地址的抽象和机制的集合。

# 3.2 Kernel address space
Xv6 maintains *one page table per process*, describing each process's user address space, *plus a single page table that describes the kernel's address space.*

kernel page table:
- one-to-one mapping (mostly)
- All physical memory
- memory-mapped I/O devices

one page table per user process

每个进程有一个页表，用于描述进程的用户地址空间，==还有一个内核地址空间，所有核心共享这个 kernel page table（所有进程共享这一个描述内核地址空间的页表）==。为了让内核使用物理内存和硬件资源，内核需要按照一定的规则排布内核地址空间，以能够确定哪个虚拟地址对应自己需要的硬件资源地址。用户地址空间不需要也不能够知道这个规则，因为用户空间不允许直接访问这些硬件资源。图 3.3 显示了此布局如何将内核虚拟地址映射到物理地址。文件 (kernel/memlayout.h) 声明了 xv6 内核内存布局的常量。

QEMU会模拟一个从0x80000000开始的RAM，一直到0x86400000。QEMU会将设备接口以控制寄存器的形式暴露给内核，通过I/O的内存映射，这些控制寄存器在0x80000000以下。kernel对这些设备接口控制寄存器的访问是直接和这些设备而不是RAM进行交互的。

内核使用“直接映射”获取 RAM 和内存映射设备寄存器；也就是说，将资源映射到与物理地址相等的虚拟地址。例如，内核本身位于虚拟地址空间和物理内存中 KERNBASE=0x80000000。直接映射简化了读取或写入物理内存的内核代码。例如，当 fork 为子进程分配用户内存时，分配器返回该内存的物理地址；当 fork 将父进程的用户内存复制到子进程时，它会直接将该地址用作虚拟地址。


![[Pasted image 20241212135831.png]]
左边和右边分别是kernel virtual address和physical address的映射关系。在虚拟地址和物理地址中，kernel都位于`KERNBASE=0x80000000`的位置，这叫做直接映射。

有一些不是直接映射的内核虚拟地址：
- trampoline page（和user pagetable在同一个虚拟地址，以便在user space和kernel space之间跳转时切换进程仍然能够使用相同的映射，真实的物理地址位于 _kernel text_ 中的`trampoline.S`），它被映射到虚拟地址空间的顶部；用户页表具有相同的映射。
- kernel stack page：每个进程有一个自己的内核栈kstack，每个kstack下面有一个没有被映射的guard page，The guard page's PTE is invalid (i.e., `PTE_V` is not set), guard page的作用是防止kstack溢出影响其他kstack。当进程运行在内核态时使用内核栈，运行在用户态时使用用户栈。**注意**：还有一个内核线程，这个线程只运行在内核态，不会使用其他进程的kstack，内核线程没有独立的地址空间。内核线程的代码应该在 _kernel text_ 中。

内核使用权限 `PTE_R` 和 `PTE_X` 映射 `trampoline` 和`kernel text`的页面。内核从这些页面读取并执行指令。内核使用权限 `PTE_R` 和 `PTE_W` 映射其他页面，以便它可以读取和写入这些页面中的内存。

# 3.3 Code: creating an address space
xv6中和页表相关的代码在`kernel/vm.c`中。最主要的结构体是`pagetable_t`，这是一个指向页表的指针。`kvm`开头的函数都是和kernel virtual address相关的，`uvm`开头的函数都是和user virtual address相关的，其他的函数可以用于这两者

几个比较重要的函数：

- `walk`：给定一个虚拟地址和一个页表，返回一个PTE指针
    
- `mappages`：给定一个页表、一个虚拟地址和物理地址，创建一个PTE以实现相应的映射
    
- `kvminit`用于创建kernel的页表，使用`kvmmap`来设置映射
    
- `kvminithart`将kernel的页表的物理地址写入CPU的寄存器`satp`中，然后CPU就可以用这个kernel页表来翻译地址了
    
- `procinit`(kernel/proc.c)为每一个进程分配(`kalloc`)kstack。`KSTACK`会为每个进程生成一个虚拟地址（同时也预留了guard pages)，`kvmmap`将这些虚拟地址对应的PTE映射到物理地址中，然后调用`kvminithart`来重新把kernel页表加载到`satp`中去。
    

每个RISC-V **CPU**会把PTE缓存到 *Translation Look-aside Buffer (TLB)* 中，当xv6更改了页表时，必须通知CPU来取消掉当前的TLB，取消当前TLB的函数是`sfence.vma()`，在`kvminithart`中被调用

The central data structure is `pagetable_t`, which is really a pointer to a RISC-V root page-table page; a `pagetable_t` may be either the kernel page table, or one of the per-process pages tables. `copyout` and `copyin` copy data to and from user virtual address provided as system call arguments;

Early in the boot sequence, `main` calls `kvminit` to create the kernel's page table using `kvmmake`. This call occurs before xv6 has enabled paging on the RISC-V, so addresses refer directly to physical memory, `kvmmake` first allocates a page of physical memory to hold the root page-table page. Then it calls `kvmmap` to install the translations that the kernel needs. The translations include the kernel's instructions and data, physical memory up to `PHYSTOP`, and memory ranges which are actually devices. `proc_mapstacks` allocates a kernel stack for each process. It calls `kvmmap` to map each stack at the virtual address generated by `KSTACK`, which leaves room for the invalid stack-guard pages.

`kvmmap` calls `mappages`, which installs mappings into a page table for a range of virtual address to a corresponding range of physical addresses. It does this separately for each virtual address in the range, at page intervals. For each virtual address to be mapped, `mappages` calls `walk` to find the address of the PTE for that address. It then initializes the PTE to hold the relevant physical page number, the desired permissions to mark the PTE as valid.

`walk` mimics the RISC-V paging hardware as it looks up the PTE for a virtual address. If the PTE isn't valid, then the required hasn't yet been allocated; if the `alloc` argument is set, `walk` allocates a new page-table page and puts its physical address in the PTE. It returns the address of the PTE in the lowest layer in the lowest layer in the tree.

`main` calls `kvminithart` to install the kernel page table.

The RISC-V has an instruction `sfence.vma` that flushes the current CPU's TLB. Xv6 executes `sfence.vma` in `kvminithart` after reloading the `satp` register, and in the trampoline code that switches to a user page table before returning to user space.

To avoid flushing the complete TLB, RISC-V CPUs may support address space identifiers (ASIDs). The kernel can then flush just the TLB entries for a particular address space.

# 3.4 Physical memory allocation for kernel
xv6对kernel space和PHYSTOP之间的物理空间在运行时进行分配，分配以页(4096 bytes)为单位。分配和释放是通过对空闲页链表进行追踪完成的，分配空间就是将一个页从链表中移除，释放空间就是将一页增加到链表中

# 3.5 Code: Physical memory allocator
Where does the allocator get  the memory to hold that data structure? It store each free page's `run` structure in the free page itself, since there's nothing else stored there.

xv6 应该通过解析硬件提供的配置信息来确定有多少物理内存可用。相反，xv6 假设机器有 128 兆字节的 RAM。kinit 调用 freerange 通过对 kfree 的逐页调用将内存添加到空闲列表中。PTE 只能引用在 4096 字节边界上对齐的物理地址（是 4096 的倍数），因此 freerange 使用 PGROUNDUP 来确保它只释放对齐的物理地址。分配器开始时没有内存；这些对 kfree 的调用给它一些内存来管理。

分配器有时会将地址视为整数，以便对其进行算术运算（例如，遍历空闲范围内的所有页面），有时会将地址用作指针来读取和写入内存（例如，操作存储在每个页面中的 run 结构）；这种地址的双重用途是分配器代码充满 C 类型转换的主要原因。另一个原因是释放和分配本质上会改变内存的类型。

函数 kfree (kernel/kalloc.c:47) 首先将要释放的内存中的每个字节设置为值 1。这将导致在释放内存后使用内存的代码（使用“悬垂引用”）读取垃圾而不是旧的有效内容；希望这会导致此类代码更快地中断。然后 kfree 将页面添加到空闲列表的前面：它将 pa 转换为指向 struct run 的指针，在 r->next 中记录空闲列表的旧开头，并将空闲列表设置为等于 r。kalloc 删除并返回空闲列表中的第一个元素。

kernel的物理空间的分配函数在`kernel/kalloc.c`中，每个页在链表中的元素是`struct run`，每个`run`存储在空闲页本身中。这个空闲页的链表`freelist`由spin lock保护，包装在`struct kmem`中。

- `kinit()`：对分配函数进行初始化，将kernel结尾到PHYSTOP之间的所有空闲空间都添加到kmem链表中，这是通过调用`freerange(end, PHYSTOP)`实现的
- `freerange()`对这个范围内所有页都调用一次`kfree`来将这个范围内的页添加到`freelist`链表中


# 3.6 User space memory
每个进程有自己的用户空间下的虚拟地址，这些虚拟地址由每个进程自己的页表维护，用户空间下的虚拟地址从0到MAXVA

当进程向xv6索要更多用户内存时，xv6先用`kalloc`来分配物理页，然后向这个进程的页表增加指向这个新的物理页的PTE，同时设置这些PTE的flag
![[Pasted image 20241212153435.png]]
图3.4是一个进程在刚刚被`exec`调用时的用户空间下的内存地址，stack只有一页，包含了`exec`调用的命令的参数从而使`main(argc, argv)`可以被执行。stack下方是一个guard page来检测stack溢出，一旦溢出将会产生一个page fault exception

`sbrk`是一个可以让进程增加或者缩小用户空间内存的system call。`sbrk`调用了`growproc`(kernel/proc.c)来改变`p->sz`从而改变**heap**中的program break，`growproc`调用了`uvmalloc`和`uvmdealloc`，前者调用了`kalloc`来分配物理内存并且通过`mappages`向用户页表添加PTE，后者调用了`kfree`来释放物理内存

Third, the kernel maps a page with trampoline code at the top of the user address space, thus a single page of physical memory shows up in all address space.

# 3.6 Code: exec
`exec`是一个system call，为以ELF格式定义的文件系统中的可执行文件创建用户空间。

`exec`先检查头文件中是否有ELF_MAGIC来判断这个文件是否是一个ELF格式定义的二进制文件，用`proc_pagetable`来为当前进程创建一个还没有映射的页表，然后用`uvmalloc`来为每个ELF segment分配物理空间并在页表中建立映射，然后用`loadseg`来把ELF segment加载到物理空间当中。注意`uvmalloc`分配的物理内存空间可以比文件本身要大。

接下来`exec`分配user stack，它仅仅分配一页给stack，通过`copyout`将传入参数的string放在stack的顶端，在ustack的下方分配一个guard page

如果`exec`检测到错误，将跳转到`bad`标签，释放新创建的`pagetable`并返回-1。`exec`必须确定新的执行能够成功才会释放进程旧的页表(`proc_freepagetable(oldpagetable, oldsz)`)，否则如果system call不成功，就无法向旧的页表返回-1

# 3.7 Real world
xv6将kernel加载到0x8000000这一RAM物理地址中，但是实际上很多RAM的物理地址都是随机的，并不一定存在0x8000000这个地址

实际的处理器并不一定以4096bytes为一页，而可能使用各种不同大小的页


# 源码剖析
## kernel/vm.c
kernel/vm.c包含了xv6中绝大部分用于==操控地址空间和页表==的代码。注意vm.c中==uvm开头的函数用来操纵用户态地址空间==，==kvm开头的函数用来操纵内核地址空间==，这是在xv6 book中已经指明的，但是要注意==它们都在内核态中运行==，使用的都是内核页表。
![[Pasted image 20241212233934.png]]

首先看第一个全局变量，==kernel_pagetable==，它是一个==pagetable_t==类型的变量。索引到对应源代码可以看到如下的定义：
![[Pasted image 20241212234050.png]]

所以在xv6的定义中，一个目录项(page table entry, pte)的大小是64比特，即8个字节，故一个4k大小的页正好对应512个PTE。事实上，采用多级页表结构的最重要的原因就是==节约因为存储页表而耗费的内存页。==我们确定要设置多少级页表时，要尽可能地保证一级虚拟地址正好映射到一个页表内，不要超出或浪费。

8 字节 PTE 的具体格式
![[Pasted image 20241212234208.png]]
相对应的，pagetable_t就是指向uint64的指针，它本质上指向了==内核页表的根目录页表(root page-table page)==的物理地址，事实上当使用MMU进行虚拟地址转换时，这个物理地址==会被存放在SATP寄存器上==，这就是第一个变量的全部含义。

再来看第二个全局变量：
![[Pasted image 20241212234322.png]]
据注释所说，etext将会==被链接脚本放置在内核代码的结束位置==，我们去看看链接脚本(kernel/kernel.ld)的对应段是怎么写的：
![[Pasted image 20241212234358.png]]
所以这里其实定义了一个字节指针，==指向内核代码部分的结束位置==

第三个全局变量是：
![[Pasted image 20241212234824.png]]

根据注释，这里的含义是它指向了trampoline代码的开始，至于==trampoline代码的实现我们会在做下一个实验的时候仔细研究一下==，这里不再展开细节。如果你打开trampoline.S这个汇编代码文件，在开头会有如下内容，它们定义了trampoline是一个全局标号，实际上==指向代码的开始==：
![[Pasted image 20241212235038.png]]
下面准备开始介绍函数了，首先从==最核心也是最底层的walk函数开始==，再逐渐延伸到它们的调用者上去，这样自底向上地介绍会更加容易理解。

## 1.2 walk 函数
walk函数的作用，如果简单来说的话就是：==用软件来模拟硬件MMU查找页表的过程==，返回以==pagetable为根页表==，经过多级索引之后==va这个虚拟地址所对应的页表项==，如果alloc != 0，则在需要时==创建新的页表页==，反之则不用。注意第一级根页表肯定是存在的，所以这个函数==最多会创建两次新的页表页==就已经到达了==叶级页表==(leaf page table)。

阅读这段代码时还需要注意一个问题，这在xv6 book中已经指出了，那就是以上代码可以正常工作的前提是，我们在==进行内核地址空间的映射==时，物理内存和虚拟内存采用的是==直接映射的方法==。
![[Pasted image 20241212235834.png]]

事实上要有这样一个概念，我们在C语言程序中所使用的指针，本质上==都是一个个虚拟地址==，那么下面这段代码看上去就有些意思了，因为我们直接将物理地址赋值给了虚拟地址，这是因为内核地址空间中执行的是==直接映射策略==。还有一种情况下，这两者也是等价的，即处理器==关闭分页机制时==，物理地址也等于虚拟地址，这种情况在下面kvmmake时会发现，到时候会再提。

![[Pasted image 20241213000057.png]]
最后，阅读这段代码时可以注意到==有几个宏非常的有意思==，我们来看一看它们的源代码，首先是宏定义==PX(level, va)==：
![[Pasted image 20241213000149.png]]
PX的含义就是抽取出虚拟地址va在==多级地址转换过程中==对应的虚拟地址字段，这个字段将==被会用来索引对应级别的页表==。level表示==当前翻译到第几级页表==，va则表示对应的虚拟地址。

接下来是PA2PTE和PTE2PA，据名字可以看出它们实现的应该是==页表项到物理地址之间的转换==：
![[Pasted image 20241213000250.png]]

## 1.3 mappages 函数
==另外一个核心函数mappages==，它是用来==装载一个新的映射关系==(可能不止一个页面大小)的，注意虚拟地址和size没必要是页对齐的，在具体实现中会使用PGROUNDDOWN这个宏来==执行自动对齐==。

但是这个函数还有一些需要格外注意的一些细节，比如==PGROUNDDOWN宏==，它和PGROUNDUP的定义一并展示如下：
![[Pasted image 20241213000826.png]]

## 1.4 kvmmap 函数
据源码注释所述这个函数负责==在内核页表中添加一个映射项==，且此函数==仅在启动时初始化内核页表时使用==。它仅仅是mappages函数薄薄的一层封装与调用，使用时==将内核页表指针传入mappages函数的第一项即可==。
![[Pasted image 20241213001155.png]]

## 1.5 kvmmake 函数
接下来，就可以看看kvmmake函数的实现了，这个函数==主要调用的就是上面的kvmmap函数==。这个函数的功能是为内核建立了一个==直接映射的页表==，如xv6 book中的示例图所示。除了trampoline和内核栈页面被映射到高地址空间以外(主要是==为了设置守护页==)，其他的部分==全部是直接映射关系==，这样的好处是方便内核的操作，==直接将返回的物理地址当虚拟地址==使用，就像上面的walk函数一样。
![[Pasted image 20241212135831.png]]
在阅读这段代码时，要注意一件事情。我们在前面所述的三个全局变量，包括==etext和trampoline全部都在映射内核页表时使用到了==，但是要注意它们本质都是虚拟地址(指针)，那么为什么可以将它们作为物理地址分配传递给kvmmap函数呢？

其实在执行上述代码的时候==xv6的分页机制是关闭的==(kernel/start.c 34-35行关闭了分页机制，这在xv6启动过程中会详细分析)，所以此代码中的所有指针，==虽然是虚拟地址，但它们本质上也等于物理地址==。如果你读得不仔细，可能领悟不到这一层的巧妙：)

## 1.6 kvminit 函数
在理解了kvmmake函数的基础上，这个函数就非常简单了，它只是==简单地调用了一下kvmmake函数==，设置好内核地址空间之后，将==返回的内核页表指针传递给了全局变量kernel_pagetable==。
![[Pasted image 20241213002343.png]]

## 1.7 kvminithart 函数
正如在上一段中所说的，其实在调用kvmmake之前在start.c中关闭了分页机制。那么什么时候再次打开分页呢，==就在这个函数kvminithart里了==。
![[Pasted image 20241213002443.png]]
函数非常简单，只有简单的两行，第一行是设置内核==根页表寄存器的==，一旦设置完毕之后==相当于也打开了分页机制==，自此之后虚拟地址就要经过MMU的翻译才可以转化为物理地址了，但是在内核态下因为大部分页面执行的还是==直接映射==，所以==物理地址和虚拟地址本质上还是相等的==(除了内核栈和trampoline页面)。MAKE_SATP是一个宏定义，定义如下：
![[Pasted image 20241213002803.png]]
要充分理解这段宏的概念，必须好好去阅读一下==RISCV的SATP寄存器各字段含义==，我已经截取了下来，如下图所示：
![[Pasted image 20241213002819.png]]
因为xv6基于RV64的体系结构，所以可以直接看==下面的RV64寄存器字段==。MODE表示使用的分页方案，xv6使用的是Sv39方案，所以==应该将MODE域设置为8==，这也就是宏==SATP_SV39的含义==，域ASID表示的是 Address Space Identifier，地址空间标识符域。这个域用来加速上下文切换过程，是==可选的==，我们看到xv6没有使用这个域，而是==直接将MODE和PPN做了或操作==得到MAKE_SATP这个宏。

再看w_satp这个函数，它接收MAKR_SATP的返回结果(其实==就是一个64位二进制数==)作为参数。它的定义如下，调用了RISCV汇编来==将x的值写入satp寄存器==，![[Pasted image 20241213003302.png]]最后再看一下sfence_vma这个函数，它的定义如下：
![[Pasted image 20241213003317.png]]
从汇编中也可以看出，这行汇编代码的作用是==完全刷新快表==，因为我们重新设置了内核页表，所以==之前的缓存必须全部清空==，这非常合理。至此，kvminithart函数结束时，==已经完成了内核地址空间的完全设置，并已经打开了分页机制==


## 1.8 walkaddr 函数
walkaddr函数是==walk函数的一层封装==，专门用来查找用户页表中==特定虚拟地址va所对应的物理地址==。所以对于本函数注意两条：
**1.它只用来查找用户页表  
2.返回的是物理地址，而非像walk函数那样只返回最终层的PTE**

最后，我试着去查看了一下到底是谁在调用这个函数，发现是==这三个函数==：
**1.copyin  
2.copyout  
3.copyinstr**
==专门负责内核态和用户态之间数据拷贝==的，

## 1.9 freewalk 函数
这个函数的作用就是==专门用来回收页表页的内存的，因为页表是多级的结构，所以此函数的实现用到了递归==，从源码上的英文注释所述，在调用这个函数时应该保证==叶子级别页表的==映射关系全部解除并释放(这将会由后面的uvmunmap函数负责)，因为此函数==专门用来回收页表页==。

uvmunmap函数和freewalk函数结合，成功实现了==页表页、物理页的全面释放==。
![[Pasted image 20241217000155.png]]

## 1.10 uvmcreate 函数
用户地址空间函数
这个函数的作用很简单，就是为用户进程==分配一个页表页并返回指向此页的指针==。

## 1.11 uvminit 函数
这个函数的作用是将initcode加载到==用户页表的0地址==上，initcode是启动第一个进程时所需要的一些代码。这个函数的作用就是将initcode映射到用户地址空间中。首先给出xv6中用户地址空间的设计，可以看到==从虚拟地址0==开始存放的是进程的代码段，对于操作系统启动的第一个进程而言，这个位置==放置的就是initcode代码==。
![[Pasted image 20241213134335.png]]
这段函数中==调用了memmove函数==，它被定义在kernel/[string].c中，实现的功能是==从src地址拷贝n个字节到dst地址==，并返回==指向目的地址的指针==

最重要的就是分类讨论，就是为了分析==潜在的Src和Dst的地址重叠问题==，重叠时反向复制(从终点开始复制到起点)，否则正向复制(从起点复制到终点)，这就是memmove函数的实现，还是非常严谨巧妙的
![[Pasted image 20241213135047.png]]

## 1.12 uvmunmap 函数
这个函数的作用是==取消用户进程页表中指定范围的映射关系==，从虚拟地址va开始释放npages个页面，但是要注意==va一定要是页对齐的==。**（这里引申思考一个问题留下，为什么在内存管理的代码中，有些代码要求传入的内存地址对齐，而有些不需要呢？）**

事实上，==uvmunmap函数和freewalk函数是组合使用的==，前面我们在看freewalk函数时还记得==它负责释放的是页表页==，那么这里的uvmunmap负责的就是释放叶级页表中PTE记录的==映射关系==，特别地，如果设置标志位do_free，此函数还会==一并将分配出去的物理页面也进行回收==。

在这段代码中使用了一个宏PTE_FLAGS，这个宏是直接用来==提取一个PTE的所有标志位的==，定义如下：
![[Pasted image 20241213140030.png]]
## 1.12 uvmdealloc
这个函数用来==回收用户页表中的页面==，将用户进程中已经分配的空间大小从==oldsz修改到newsz==，并==返回新地址空间的大小==，值得注意的是oldsz不一定大于newsz，也就是说这个函数==不是一定会导致用户地址空间缩小的==。
**事实上，这个函数和sbrk系统调用有直接关系，它是用来回收多余的进程堆内存空间的，详见growproc函数(kernel/proc.c)。**

## 1.13 uvmalloc 函数
一个函数来为用户进程==向内核申请更多的内存==

## 1.14 uvmcopy 函数
uvmcopy函数是为==fork系统调用==服务的，它会将父进程的整个地址空间==全部复制到子进程中==，这包括==页表本身==和==页表指向的物理内存中的数据==。

## 1.15 uvmclear 函数
uvmclear函数专门用来清除一个PTE的==用户使用权限==，用于exec函数来设置守护页。实现非常简单，如下所示：

## 1.16 uvmfree 函数
还记得我们之前所说的uvmunmap用于==取消叶级页表的映射关系==，freewalk==用于释放页表页==，两者结合可以完全释放内存空间吗。uvmfree函数就是两者的一个简单结合和封装，用于==完全释放用户的地址空间==。

## 1.17 copyin 函数
这个函数负责从**给定的用户页表pagetable的虚拟地址srcva处拷贝长度为len字节的数据到地址dst处**。在阅读这段代码时，思考一个问题：==为什么内核目的地址dst使用指针来表示，而用户态的地址却使用uint64 srcva来代替？==

因为，copyin代码会==运行在内核态下==，所以在copyin代码中凡是引用指针变量的地方(如dst)都会通过MMU硬件单元查询==内核页表==翻译对应的物理地址。

而对于用户态下的虚拟地址，只能够用软件模拟 MMU 的查找过程，也就是在copyin代码中调用walkaddr的原因，它本质上是用==软件逻辑实现了硬件MMU的地址翻译过程==。

## 1.18 copyinstr 函数

## 1.19 copyout 函数
从内核空间拷贝数据到用户空间