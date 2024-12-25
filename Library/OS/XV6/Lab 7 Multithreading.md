# Uthread: switching between threads
In this exercise you will design the context switch mechanism for a user-level threading system, and then implement it. 

Your job is to come up with a plan to create threads and save/restore registers to switch between threads, and implement that plan.

One goal is ensure that when `thread_schedule()` runs a given thread for the first time, the thread executes the funtion passed to `thread_create()`, on its own stack, on its own stack. Another goal is to ensure that `thread_switch` saves the registers of the thread being switched away from, restores the registers of the thread being switched to, and returns to the point in the latter thread's instructions where it last left off.

Some hints:
- `threa_switch` needs to save/restore only the callee-save registers. Why?

# Using threads 
In this assignment you will explore parallel programming with threads and locks using a hash table. 

This assignment uses the UNIX `pthread` threading library. 

The file `ph.c` contains a simple hash table that is correct if used from a single thread, but incorrect when used from multiple threads.

ph 的参数指定在哈希表上执行 put 和 get 操作的线程数。运行一段时间后，ph 1 将产生类似以下的输出：
![[Pasted image 20241224151914.png]]
ph 运行两个基准测试。首先，它通过调用 put() 将大量键添加到哈希表中，并打印每秒实现的放入次数。然后，它使用 get() 从哈希表中获取键。它打印由于放入而应该在哈希表中但缺失的键数（在本例中为零），并打印每秒实现的获取次数。

这里的优化思路，也是多线程效率的一个常见的优化思路，==就是降低锁的粒度==。由于哈希表中，不同的 bucket 是互不影响的，一个 bucket 处于修改未完全的状态并不影响 put 和 get 对其他 bucket 的操作，所以实际上只需要确保两个线程不会同时操作同一个 bucket 即可，并不需要确保不会同时操作整个哈希表。

所以可以将加锁的粒度，从整个哈希表一个锁降低到每个 bucket 一个锁。


# Barrier
A barrier: a point in an application at which all participating threads must wait until all other participating threads reach that point too. 

In each loop iteration a thread calls `barrier()` and then sleeps for a random number of microseconds. The assert triggers, because one thread leaves the barrier before the other thread has reached the barrier. The desired behavior is that each thread blocks in `barrier()` until all `nthreads` of them have called `barrier()`.
![[Pasted image 20241224155354.png]]

利用 pthread 提供的条件变量方法
![[Pasted image 20241224160059.png]]
线程进入同步屏障 barrier 时，将已进入屏障的线程数量增加 1，然后再判断是否已经达到总线程数。  
如果未达到，则进入睡眠，等待其他线程。  
如果已经达到，则唤醒所有在 barrier 中等待的线程，所有线程继续执行；屏障轮数 + 1；

「将已进入屏障的线程数量增加 1，然后再判断是否已经达到总线程数」这一步并不是原子操作，并且这一步和后面的两种情况中的操作「睡眠」和「唤醒」之间也不是原子的，如果在这里发生 race-condition，则会导致出现 「lost wake-up 问题」（线程 1 即将睡眠前，线程 2 调用了唤醒，然后线程 1 才进入睡眠，导致线程 1 本该被唤醒而没被唤醒）

解决方法是，「屏障的线程数量增加 1；判断是否已经达到总线程数；进入睡眠」这三步必须原子。所以使用一个互斥锁 barrier_mutex 来保护这一部分代码。pthread_cond_wait 会在进入睡眠的时候原子性的释放 barrier_mutex，从而允许后续线程进入 barrier，防止死锁。

  
