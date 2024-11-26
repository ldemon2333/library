`free` 是 Linux 系统中用于查看系统内存使用情况的命令。它显示了物理内存、交换空间（swap）、以及各个内存区域的使用情况。常用的 `free` 命令可以提供系统当前的内存状况，帮助你了解内存是否充足，是否需要优化。

### 基本用法

```bash
free
```

这会显示一个简单的内存使用报告，包括：

- **total**：总内存大小。
- **used**：已使用的内存。
- **free**：空闲内存。
- **shared**：多个进程共享的内存大小。
- **buffers/cache**：缓存和缓冲区占用的内存。
- **available**：系统当前可用的内存（注意，这个值是计算了缓存和缓冲区释放后的可用内存）。

### 示例输出

```bash
             total       used       free     shared    buffers     cached
Mem:       16345608   12345678    4000000    1234567    2000000    3000000
-/+ buffers/cache:    5000000    11345608
Swap:      8388608    2000000    6388608
```

### 各列解释

- **Mem**：物理内存的使用情况。
    
    - **total**：系统总内存。
    - **used**：已使用的内存。
    - **free**：空闲内存。
    - **shared**：由多个进程共享的内存（一般来说，文件缓存会占用一些共享内存）。
    - **buffers**：内核使用的缓冲区内存。
    - **cached**：用作缓存的内存。
- **-/+ buffers/cache**：这行显示了去除缓冲区和缓存后的“真正”的已用内存和可用内存。这是衡量系统实际内存使用的重要指标。
    
- **Swap**：交换空间的使用情况。
    
    - **total**：交换空间总大小。
    - **used**：已使用的交换空间。
    - **free**：空闲的交换空间。

### 常用选项

1. **`-h`**：以人类可读的方式显示（自动转换为 KB、MB 或 GB）。
    
    ```bash
    free -h
    ```
    
    示例：
    
    ```bash
                 total        used        free      shared  buff/cache   available
    Mem:           16Gi       11Gi       3.8Gi       1.1Gi       1.7Gi       4.7Gi
    Swap:         8.0Gi       1.9Gi       6.1Gi
    ```

# Name 
free - Display amount of free and used memory in the system

# Description
free displays the total amount of free and used physical and swap memory in the system, as well as the buffers and caches used by the kernel. The information is gathered by parsing /proc/meminfo. The displayed columns are:

total   Total install memory 

buffers: Memory used by kernel buffers (Buffers in /proc/meminfor)

cache: Memory used by the page cache and slabs (Cached and SReclaimable in /proc/meminfo)

buff/cache: Sum of buffers and cachef

