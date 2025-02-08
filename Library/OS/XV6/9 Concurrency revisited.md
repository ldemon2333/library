# 9.1 Locking patterns
It's vital that a given disk block have at most one copy in the cache; otherwise, different processes might make conflicting changes to different copies of what ought to be the same block. This is a common pattern: one lock for the set of items, plus one lock per item.

Ordinarily the same function that acquires a lock will release it. But a more precise way to view things is tha a lock is acquired at the start of a sequence that must appear atomic, and released when that sequence ends. Another example is the `acquiresleep` in `ilock`; this code often sleeps while reading the disk; it may wake up on a different CPU, which means the lock may be acquired and released on different CPUs.

# 9.2 Lock-like patterns
While in each case a lock protects the flag or reference count, it is the latter that prevents the object from being permaturely freed.

However, `namex` release each lock at the end of the loop, since if it held multiple locks it could deadlock with itself it the pathname included a dot.
There are two lessons here: a data object may be protected from concurrency in different ways at different points in its lifetime, and the protection may take the form of implicit structure rather than explicit locks.

# 9.3 No locks at all


# 9.4 Parallelism
