How to decide which page to evict?
How can the OS decide which page (or pages) to evict from memory? 

# 22.1 Cache Management
knowing the number of cache hits and misses let us calculate the **average memory access time (AMAT)**  for a program. 
![[Pasted image 20241127142609.png]]
where Tm represents the cost of accessing memory, Td the cost of accessing disk, and Pmiss  the probability of not finding the data in the cache (a miss);

# 22.2 The Optimal Replacement Policy

# 22.3 A simple Policy: FIFO

# 22.4 Another Simple Policy: Random

# 22.5 Using History: LRU

# 22.11 抖动
Some current systems take more a draconian approach to memory overload. For example, some versions of Linux run an **out-of-memory killer** when memory is oversubscribed; this daemon chooses a memory-intensive process and kills it, thus reducing memory in a none-too-subtle manner. While successful at reducing memory pressure, this approach can have problems, if, for example, it kills the X server and thus renders any applications requiring the display unusable.

# 22.12 Summary
buy more memory
