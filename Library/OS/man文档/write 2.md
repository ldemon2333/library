# NAME
write - write to a file descriptor
# SYNOPSIS
```
#include <unistd.h>
ssize_t write(int fd, const void *buf, size_t count);
```

# DESCRIPTION
write() writes up to count bytes from the buffer starting at buf to the file referred to by the file descriptor fd.

例如，如果底层物理介质上的空间不足，或者遇到了 RLIMIT_FSIZE 资源限制（请参阅 setrlimit(2)），或者在写入少于 count 个字节后调用被信号处理程序中断，则写入的字节数可能小于 count 个。（另请参阅 [[pipe 7]]）

For a seekable file (i.e., one to which [[lseek 2]] may be applied, for example, a regular file) writing takes place at the file offset, and the file offset is incremented by the number of bytes actually written. If the file was [[open 2]]ed with O_APPEND, 则在写入之前，首先将文件偏移量设置为文件末尾。文件偏移量的调整和写入操作作为原子步骤执行。

POSIX 要求，如果可以证明 [[read 2]] 在 write() 返回后发生，则该 [[read 2]] 将返回新数据。请注意，并非所有文件系统都符合 POSIX 要求。

# RETURN VALUE
On success, the number of bytes written is returned. On error, -1 is returned, and errno is set to indicate the cause of the error.

请注意，成功的 write() 可能会传输少于 count 个字节。这种部分写入可能由于各种原因而发生；例如，因为磁盘设备上的空间不足以写入所有请求的字节，或者因为阻塞的对套接字、管道或类似对象的 write() 在传输了一些字节之后但在传输完所有请求的字节之前被信号处理程序中断。如果发生部分写入，调用者可以进行另一个 write() 调用来传输剩余的字节。后续调用将传输更多字节或可能导致错误（例如，如果磁盘现在已满）。

---
`write()` 系统调用是 Linux 和类 Unix 操作系统中用于向文件描述符写入数据的核心操作。它可以用于写入文件、管道、套接字、设备等。与 `read()` 相对，`write()` 是进行输出操作的基本接口。

### `write()` 系统调用的原型

```c
ssize_t write(int fd, const void *buf, size_t count);
```

#### 参数：
- **`fd`**：文件描述符，表示目标文件或设备。可以是一个普通文件、管道、套接字、标准输出（`STDOUT_FILENO`）等。
- **`buf`**：指向数据的指针，这些数据将被写入到由 `fd` 指定的文件。
- **`count`**：要写入的字节数。

#### 返回值：
- 成功时，返回实际写入的字节数（可能小于请求写入的字节数）。
- 如果发生错误，返回 `-1`，并设置 `errno` 变量来指示错误类型。

### 工作原理
- `write()` 系统调用将指定的缓冲区 `buf` 中的 `count` 个字节写入到文件描述符 `fd` 指定的文件或设备中。
- 对于文件描述符 `fd`，可以是一个普通的文件、设备文件、管道、套接字等。
- `write()` 是一种低级的 I/O 操作，它直接与操作系统内核交互。
- 写入操作不会自动进行缓冲区管理（这通常由高级库函数（如 `fwrite()`）管理）。系统调用会直接向内核的文件系统写入数据。

### `write()` 示例代码

#### 1. 向文件写入数据

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

int main() {
    const char *msg = "Hello, World!";
    int fd;

    // 打开文件用于写操作
    fd = open("example.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd == -1) {
        perror("open");
        return -1;
    }

    // 使用 write() 写数据
    ssize_t bytesWritten = write(fd, msg, strlen(msg));
    if (bytesWritten == -1) {
        perror("write");
        close(fd);
        return -1;
    }

    printf("Successfully wrote %zd bytes to file\n", bytesWritten);

    // 关闭文件
    close(fd);

    return 0;
}
```

#### 2. 向管道写入数据（父子进程通信）

```c
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <fcntl.h>
#include <stdlib.h>

int main() {
    int pipefd[2];
    pid_t pid;
    const char *msg = "Hello from parent process";

    // 创建管道
    if (pipe(pipefd) == -1) {
        perror("pipe");
        return -1;
    }

    pid = fork();
    if (pid == -1) {
        perror("fork");
        return -1;
    }

    if (pid == 0) { // 子进程
        close(pipefd[1]); // 关闭写端

        char buf[128];
        ssize_t bytesRead = read(pipefd[0], buf, sizeof(buf));
        if (bytesRead == -1) {
            perror("read");
            exit(1);
        }

        buf[bytesRead] = '\0'; // null-terminate the string
        printf("Child received: %s\n", buf);

        close(pipefd[0]);
    } else { // 父进程
        close(pipefd[0]); // 关闭读端

        ssize_t bytesWritten = write(pipefd[1], msg, strlen(msg) + 1); // 包括 null 终止符
        if (bytesWritten == -1) {
            perror("write");
            exit(1);
        }
        close(pipefd[1]); // 关闭写端
    }

    return 0;
}
```

### `write()` 常见行为和注意事项：

1. **阻塞行为**：
   - 对于管道和套接字等文件描述符，`write()` 可能会被阻塞，直到有足够的空间进行写入。例如，在管道缓冲区满时，写操作会阻塞，直到缓冲区有空间可用。
   - 对于常规文件，`write()` 通常不会阻塞，除非文件系统遇到问题。

2. **部分写入**：
   - `write()` 每次调用可能不会写入请求的所有字节。返回值表示实际写入的字节数，通常情况下它小于 `count`。这意味着 `write()` 调用可能会部分完成数据写入。你可能需要多次调用 `write()` 来完成所有数据的写入，特别是在处理大文件时。

3. **文件描述符**：
   - 文件描述符 `fd` 必须是有效的，且指向可以写入的资源。对于打开的文件，它通常是通过 `open()` 获取的文件描述符。

4. **错误处理**：
   - `write()` 调用失败时，返回 `-1`，并通过 `errno` 变量设置具体的错误类型。常见的错误包括文件描述符无效、权限不足等。

5. **标准输出和标准错误输出**：
   - 对于文件描述符 `STDOUT_FILENO`（标准输出）和 `STDERR_FILENO`（标准错误输出），`write()` 会直接向终端或控制台输出数据。

6. **`O_APPEND` 标志**：
   - 如果使用 `open()` 打开文件时使用了 `O_APPEND` 标志，`write()` 会将数据追加到文件的末尾，而不是覆盖现有内容。

### 常见的 `errno` 错误：
- **EBADF**：提供的文件描述符无效（未打开或者关闭的文件描述符）。
- **EFAULT**：`buf` 参数是无效的内存地址。
- **EINTR**：系统调用被信号中断，可以重新调用 `write()`。
- **EAGAIN / EWOULDBLOCK**：非阻塞模式下，写操作无法立即完成（通常发生在管道或套接字缓冲区满时）。

### `write()` 与其他 I/O 系统调用的比较：
- **`write()` vs. `fwrite()`**：`write()` 是系统调用，直接与内核交互，适用于低级文件操作。`fwrite()` 是 C 标准库函数，提供缓冲区管理，适用于流式写入。
- **`write()` vs. `send()`**：`send()` 是用于网络套接字的写操作，常用于 TCP/UDP 数据发送。`write()` 更通用，可以用于文件、设备等多种类型的文件描述符。

### 使用场景：
- **文件写入**：写入普通文件、日志文件等。
- **管道通信**：通过管道在进程之间传递数据。
- **设备控制**：向设备文件写入数据，例如向硬件设备发送命令。
- **套接字通信**：通过网络套接字向远程主机发送数据。