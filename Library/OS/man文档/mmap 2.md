在 Linux 中，`mmap` 是一种用于将文件或设备映射到内存的系统调用。它可以显著提高文件访问效率，因为它允许程序直接通过内存地址操作文件，而不需要传统的读写操作。

`mmap()` creates a new mapping in the virtual address space of the calling process. The starting address for the new mapping is specified in addr. The length argument specifies the length of the mapping (which must be greater than 0).

If addr is NULL, then the kernel chooses the (page-aligned) address at which to create the mapping; this is the most portable method of creating a new mapping.

---

### 基本概念

`mmap` 的核心思想是将文件内容映射到进程的虚拟内存地址空间，使得文件的内容像普通内存一样可以通过指针直接访问。

- **用途**：
  1. 高效读取和写入文件，特别适用于大文件的部分操作。
  2. 文件共享，不同进程间可以通过共享同一映射实现高效通信。
  3. 内存分配，利用匿名映射（不绑定文件）管理内存。

- **优点**：
  1. 减少系统调用次数，提高性能。
  2. 自动缓存文件数据，避免重复 IO 操作。

- **缺点**：
  1. 映射的大小受限于进程的虚拟内存空间。
  2. 操作时需要小心内存同步问题。

---

### `mmap` 的系统调用

#### 函数签名
```c
#include <sys/mman.h>

void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
```

#### 参数说明
- **`addr`**: 指定映射的起始地址（通常为 `NULL`，表示由系统决定）。
- **`length`**: 映射的大小（以字节为单位）。
- **`prot`**: 映射区域的权限：
  - `PROT_READ`: 可读。
  - `PROT_WRITE`: 可写。
  - `PROT_EXEC`: 可执行。
  - `PROT_NONE`: 无访问权限。
- **`flags`**: 控制映射的行为：
  - `MAP_SHARED`: 映射的区域可被多个进程共享。
  - `MAP_PRIVATE`: 写时复制，不会影响原始文件。
  - `MAP_ANONYMOUS`: 创建匿名映射，与文件无关。
- **`fd`**: 文件描述符（匿名映射时设为 `-1`）。
- **`offset`**: 文件映射的起始偏移量，必须是页大小的倍数。

#### 返回值
- 成功：返回映射区域的首地址。
- 失败：返回 `MAP_FAILED` (`(void *)-1`)，并设置 `errno`。

---

### 使用示例

#### 1. 映射文件到内存
```c
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

int main() {
    int fd = open("example.txt", O_RDONLY); // 打开文件
    if (fd < 0) {
        perror("open");
        return 1;
    }

    // 获取文件大小
    off_t size = lseek(fd, 0, SEEK_END);

    // 映射文件到内存
    char *data = mmap(NULL, size, PROT_READ, MAP_PRIVATE, fd, 0);
    if (data == MAP_FAILED) {
        perror("mmap");
        close(fd);
        return 1;
    }

    // 打印文件内容
    write(STDOUT_FILENO, data, size);

    // 释放映射
    munmap(data, size);
    close(fd);
    return 0;
}
```

---

#### 2. 创建匿名内存映射
匿名映射可以用作进程间共享内存。

```c
#include <sys/mman.h>
#include <stdio.h>

int main() {
    // 创建匿名映射
    size_t size = 4096; // 映射 4KB
    void *addr = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_ANONYMOUS | MAP_SHARED, -1, 0);
    if (addr == MAP_FAILED) {
        perror("mmap");
        return 1;
    }

    // 写入数据
    char *data = (char *)addr;
    sprintf(data, "Hello, mmap!");

    // 读取数据
    printf("Data: %s\n", data);

    // 释放映射
    munmap(addr, size);
    return 0;
}
```

---

### 常见问题

1. **如何确定映射区域的大小？**
   - 映射的大小必须是页面大小的整数倍。可以使用 `getpagesize()` 或 `sysconf(_SC_PAGESIZE)` 获取页面大小。

2. **文件大小小于映射大小？**
   - 如果映射超过文件大小，访问超出部分会导致 `SIGBUS` 错误。

3. **同步文件和内存的内容？**
   - 使用 `msync()` 确保内存内容写回文件。

---

### 调试和优化

- 使用 `strace` 查看 `mmap` 系统调用：
  ```bash
  strace ./program
  ```

- 检查内存映射的状态：
  ```bash
  cat /proc/<pid>/maps
  ```

`mmap` 是 Linux 下非常强大的文件和内存操作工具，灵活运用它可以显著提升性能。