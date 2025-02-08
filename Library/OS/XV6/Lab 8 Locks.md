A common symptom of poor parallelism on multi-core machines is high lock contention.

# Memory allocator 
The program `user/kalloctest` stresses xv6's memory allocator: three processes grow and shrink their address spaces, resulting in many calls to `kalloc` and `kfree`

The basic idea is to maintain a free list per CPU, each list with its own lock. Allocations and frees on different CPUs can run in parallel. The main challenge will be to deal with the case in which one CPU's free list is empty, but another CPU's list has free memory; in that case, the one CPU must "steal" part of the other CPU's free list. Stealing may introduce lock contention, but that will hopefully be infrequent.

这里体现了一个先 profile 再进行优化的思路。如果一个大锁并不会引起明显的性能问题，有时候大锁就足够了。只有在万分确定性能热点是在该锁的时候才进行优化，「过早优化是万恶之源」。

这里解决性能热点的思路是「将共享资源变为不共享资源」。锁竞争优化一般有几个思路：

- 只在必须共享的时候共享（对应为将资源从 CPU 共享拆分为每个 CPU 独立）
- 必须共享时，尽量减少在关键区中停留的时间（对应“大锁化小锁”，降低锁的粒度）

该 lab 的实验目标，即是为每个 CPU 分配独立的 freelist，这样多个 CPU 并发分配物理页就不再会互相排斥了，提高了并行性。

但由于在一个 CPU freelist 中空闲页不足的情况下，仍需要从其他 CPU 的 freelist 中“偷”内存页，所以一个 CPU 的 freelist 并不是只会被其对应 CPU 访问，还可能在“偷”内存页的时候被其他 CPU 访问，故仍然需要使用单独的锁来保护每个 CPU 的 freelist。但一个 CPU freelist 中空闲页不足的情况相对来说是比较稀有的，所以总体性能依然比单独 kmem 大锁要快。在最佳情况下，也就是没有发生跨 CPU “偷”页的情况下，这些小锁不会发生任何锁竞争。

# Buffer cache
If multiple processes use the file system intensively, they will likely contend for `bcache.lock`, which protects the disk block cache in kernel/bio.c

Modify the block cache so that the number of `acquire` loop iterations for all locks in the bcache is close to zero when running `bcachetest`

Hints:
- It is OK to use a fixed number of buckets and not resize the hash table dynamically. Use a prime number of buckets (13) to reduce the likelihood of hashing conflits.
- Searching in the hash table for a buffer and allocating an entry for that buffer when the buffer is not found must be atomic.
- It is OK to serialize eviction in `bget`.
- During eviction you may need to hold the bcache lock and a lock bucket. 
- Remove the list of all buffers and instead time-stamp buffers using the time of their last use. With this change `brelse` doesn't need to acquire the bcache lock, and `bget` can select the least-recently used block based on the time-stamps.

在这里， bcache 属于“必须共享”的情况，所以需要用到第二个思路，降低锁的粒度，用更精细的锁 scheme 来降低出现竞争的概率。

新的改进方案，可以**建立一个从 blockno 到 buf 的哈希表，并为每个桶单独加锁**。这样，仅有在两个进程同时访问的区块同时哈希到同一个桶的时候，才会发生锁竞争。当桶中的空闲 buf 不足的时候，从其他的桶中获取 buf。
  
