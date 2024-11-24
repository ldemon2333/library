# SYNOPSIS
```
#include <unistd.h>
int close(int fd);
```

# DESCRIPTION
close() closes a file descriptor, so that it no longer refers to any file and may be reused. Any record locks (see [[fcntl 2]]) held on the file it was associated with, and owned by the process, are removed (regardless of the file descriptor that was used to obtain the lock).

If fd is the last file descriptor referring to the underlying open file description, the resources associated with the open file description are freed; if the file descriptor was the last reference to a file which has been removed using [[unlink 2]], the file is deleted.

# NOTES
A successful close does not guarantee that the data has been successfully saved to disk, as the kernel uses the buffer cache to defer writes. Typically, filesystems do not flush buffers when a file is closed. If you need to be sure that data is physically stored on the underlying disk, use [[fsync 2]]. (It will depend on the disk hardware at this point).

The close-on-exec file descriptor flag can be used to ensure that a file descriptor is automatically closed upon a successful [[execve 2]]; see [[fcntl 2]] for details.

It is probably unwise to close file descriptors while they may be in use by system calls in other threads in the same process. Since a file descriptor may be reused, there are some obscure race conditions that may cause unintended side effects.


---
在操作系统中，`close` 系统调用用于关闭文件描述符，使得程序不再能访问该文件。以下是 `close` 的详细解释：

### 基本概念
- 文件描述符是一个整数，它表示一个文件（或设备）的打开句柄。每个进程会维护一个文件描述符表，记录当前已打开的文件。
- 在调用 `open`、`socket` 等系统调用打开一个文件或网络连接时，操作系统会返回一个文件描述符。
- `close` 会告诉操作系统释放与该文件描述符相关的资源，包括文件表项、内核缓冲区等。

### `close` 的工作原理
1. **释放文件描述符**：`close(fd)` 会在文件描述符表中移除 `fd` 条目，使得 `fd` 无效。
2. **减少引用计数**：如果多个文件描述符指向同一个文件（例如通过 `dup` 或 `fork`），调用 `close` 只会减少引用计数，并不会直接关闭文件。只有当引用计数降为 0 时，系统才会真正释放文件资源。
3. **同步数据到磁盘**：对于文件的写操作，`close` 通常会将内存中的缓冲数据同步到磁盘上，确保数据安全。

### 返回值
- 成功时，`close` 返回 0。
- 失败时，返回 -1，并设置 `errno` 以提供具体的错误信息。例如，如果 `fd` 已经关闭或无效，则会返回 `EBADF` 错误码。

### 常见错误
- **多次关闭**：重复关闭同一个文件描述符会导致 `EBADF` 错误。
- **使用关闭后的描述符**：关闭一个文件描述符后，重新使用它会导致不可预测的错误（例如，系统可能会将相同的文件描述符分配给不同的文件）。
- **资源泄漏**：忘记调用 `close` 会导致文件描述符泄漏，最终可能耗尽系统的文件描述符，影响程序和系统的正常运行。

### 示例
```c
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>

int main() {
    int fd = open("example.txt", O_RDONLY);
    if (fd == -1) {
        perror("Error opening file");
        return 1;
    }
    
    // 读取和处理文件内容
    
    if (close(fd) == -1) {
        perror("Error closing file");
        return 1;
    }
    
    return 0;
}
```

在此示例中，我们打开文件 `example.txt` 并在完成操作后使用 `close` 关闭文件描述符 `fd`。

### 总结
`close` 是一种资源管理机制，可以防止文件描述符泄漏。它对于保持系统资源稳定和确保数据完整性至关重要。