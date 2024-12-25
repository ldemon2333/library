虚拟内存提供了间接层：内核可以通过将 PTE 标记为无效或只读来拦截内存引用，从而导致页面错误，并且可以通过修改 PTE 来更改地址的含义。计算机系统中有句俗语：任何系统问题都可以通过间接层来解决。

# The solution
The goal of copy-on-write `fork()` is to defer allocating and copying physical memory pages for the child until the copies are actually needed, if ever.

COW `fork()` creates just a pagetable for the child, with PTEs for user memory pointing to the parent's physical pages. COW fork() marks all the PTEs in both parent and child are not writable. When either process tries to write one of these COW pages, the CPU will force a page fault. 

COW `fork()` makes freeing of the physical pages that implement user memory a little tricker. A given physical page may be referred to by multiple processes' page tables, and should be freed only when the last reference disappears.

# hints
1. Modify `uvmcopy()` to map the parent's physical pages into the child, instead of allocating new pages. Clear `PTE_W` in the PTEs of both child and parent.
2. Modify `usertrap()` to recognize page faults. When a page-fault occurs on a COW page, allocate a new page with `kalloc()`, copy the old page to the new page, and install the new page in the PTE with `PTE_W` set.
3. Ensure that each physical page is freed when the last PTE reference to it goes away -- but not before. For each physical page, a "reference count" of the number of user page tables that refer to that page.
4. Modify `copyout()` to use the same scheme as page faults when it encounters a COW page.
5. It may be useful to have a way to record, for each PTE, whether it is a COW mapping. You can use the RSW (reserved for software) bits in the RISC-V PTE for this.
6. If a COW page fault occurs and there's no free memory, the process should be killed.

# fork 时不立刻复制内存
首先修改`uvmcopy()`，在复制父进程的内存到子进程的时候，不立刻复制数据，而是建立指向原物理页的映射，并将父子两端的页表项都设置为不可写。
![[Pasted image 20241219220000.png]]

如果父进程有一个页面是只读的，意味着 PTE_R，直接共享物理页，不用设置标志COW，设置标志 COW 是为了写的时候触发page fault，复制新的物理页完成写操作。而子进程PTE_R 继承后不能设置标志 COW，也不能写，直接共享同一个物理页。

这样，fork 时就不会立刻复制内存，只会创建一个映射了。这时候如果尝试修改懒复制的页，会出现 page fault 被 usertrap() 捕获。接下来需要在 usertrap() 中捕捉这个 page fault，并在尝试修改页的时候，执行实复制操作。

# 捕获写操作并执行复制
在 usertrap() 中添加对 page fault 的检测，并在当前访问的地址符合懒复制页条件时，对懒复制页进行实复制操作：
![[Pasted image 20241219220748.png]]

scause = 12 是指令page fault，而指令page 一般是只读页，这里的page fault是写的时候发生的，所以 scause=13 || scause = 15
![[Pasted image 20241216230323.png]]
![[Pasted image 20241219220956.png]]
![[Pasted image 20241219221103.png]]
同时 copyout() 由于是软件访问页表，不会触发缺页异常，所以需要手动添加同样的监测代码（同 lab5），检测接收的页是否是一个懒复制页，若是，执行实复制操作：
![[Pasted image 20241219222302.png]]
接下来，还需要做页的生命周期管理，确保在所有进程都不使用一个页时才将其释放

# 物理页生命周期以及引用计数
支持了懒分配后，由于一个物理页可能被多个进程（多个虚拟地址）引用，并且必须在最后一个引用消失后才可以释放回收该物理页，所以一个物理页的生命周期内，现在需要支持以下操作：

- kalloc(): 分配物理页，将其引用计数置为 1
- krefpage(): 创建物理页的一个新引用，引用计数加 1
- kcopy_n_deref(): 将物理页的一个引用实复制到一个新物理页上（引用计数为 1），返回得到的副本页；并将本物理页的引用计数减 1
- kfree(): 释放物理页的一个引用，引用计数减 1；如果计数变为 0，则释放回收物理页

一个物理页 p 首先会被父进程使用 kalloc() 创建，fork 的时候，新创建的子进程会使用 krefpage() 声明自己对父进程物理页的引用。当尝试修改父进程或子进程中的页时，kcopy_n_deref() 负责将想要修改的页实复制到独立的副本，并记录解除旧的物理页的引用（引用计数减 1）。最后 kfree() 保证只有在所有的引用者都释放该物理页的引用时，才释放回收该物理页。

  这里首先定义一个数组 pageref[] 以及对应的宏，用于记录与获取某个物理页的引用计数：
  ![[Pasted image 20241219222740.png]]
  ![[Pasted image 20241219222837.png]]
  ![[Pasted image 20241219222859.png]]
  ![[Pasted image 20241219222925.png]]
  ![[Pasted image 20241219222955.png]]
这里可以看到，为 pageref[] 数组定义了自旋锁 pgreflock，并且在除了 kalloc 的其他操作中，都使用了 `acquire(&pgreflock);` 和 `release(&pgreflock);` 获取和释放锁来保护操作的代码。这里的锁的作用是防止竞态条件（race-condition）下导致的内存泄漏。

举一个很常见的 fork() 后 exec() 的例子：
![[Pasted image 20241219223146.png]]

在这一个执行流过后，最终结果是物理页 p 并没有被释放回收，然而父进程和子进程都已经丢弃了对 p 的引用（页表中均没有指向 p 的页表项），这样一来 p 占用的内存就属于泄漏内存了，永远无法被回收。

加了锁之后，保证了这种情况不会出现。

注意 kalloc() 可以不用加锁，因为 kmem 的锁已经保证了同一个物理页不会同时被两个进程分配，并且在 kalloc() 返回前，其他操作 pageref() 的函数也不会被调用，因为没有任何其他进程能够在 kalloc() 返回前得到这个新页的地址。当然，保险起见加锁也没问题。

加锁对全局数组 pgref[] 进行互斥访问，这是全局共享数据，加锁保护。