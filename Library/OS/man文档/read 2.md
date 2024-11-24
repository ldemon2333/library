# NAME 
read - read from a file descriptor

# SYNOPSIS
```
#include <unistd.h>
ssize_t read(int fd, void *buf, size_t count);
```

# DESCRIPTION
read() attempts to read up to count bytes from file descriptor fd into the buffer starting at buf.

对于支持查找的文件，读取操作从文件偏移量开始，并且文件偏移量会按读取的字节数递增。如果文件偏移量位于文件末尾或超出文件末尾，则不会读取任何字节，read() 将返回
零。

If count is zero, read() may detect the errors described below

# RETURN VALUE
成功时，将返回读取的字节数（零表示文件末尾），并且文件位置将按此数字前进。如果此数字小于请求的字节数，则不是错误；例如，这可能是因为现在实际可用的字节较少（可能是因为我们接近文件末尾，或者因为我们正在从管道或终端读取），或者因为 read() 被信号中断。

On error, -1 is returned, and errno is set appropriately. In this case, it is left unspecified whether the file position (if any) changes.

# NOTES
The types size_t and ssize_t are, respectively, unsigned and signed integer data types specified by POSIX.1.

---
`read()` 系统调用是 Linux 和类 Unix 操作系统中最基本的输入操作之一，用于从文件描述符中读取数据。它是文件操作接口的一部分，可以读取普通文件、设备文件、管道、套接字等中的数据。

### `read()` 系统调用的原型

```c
ssize_t read(int fd, void *buf, size_t count);
```

#### 参数：
- `fd`：文件描述符，表示要读取数据的文件。可以是一个普通文件、管道、FIFO、套接字等的文件描述符。
- `buf`：指向一个缓冲区的指针，`read()` 会将数据从文件中读取到这个缓冲区中。
- `count`：要读取的最大字节数。

#### 返回值：
- 成功时，返回读取的字节数。如果返回值是 0，表示文件结束（EOF）。
- 如果发生错误，返回 `-1`，并且会设置 `errno` 变量来指示错误类型。

### 工作原理
- `read()` 系统调用从指定的文件描述符中读取最多 `count` 个字节的数据，并将它们存储到 `buf` 指向的缓冲区中。
- **文件描述符** 是一种表示文件的整数值，通常通过 `open()` 获取。
- 如果缓冲区大小大于文件实际数据的大小，`read()` 将只读取文件中的数据，不会导致内存溢出。
- 如果文件末尾被读取完，`read()` 返回 0。

### `read()` 示例代码

#### 1. 读取普通文件中的内容

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>

int main() {
    int fd;
    char buf[128];
    ssize_t bytesRead;

    // 打开一个文件（只读模式）
    fd = open("example.txt", O_RDONLY);
    if (fd == -1) {
        perror("open");
        return -1;
    }

    // 读取文件内容
    bytesRead = read(fd, buf, sizeof(buf) - 1); // 最多读取 127 字节
    if (bytesRead == -1) {
        perror("read");
        close(fd);
        return -1;
    }

    // 以字符串形式输出读取的数据
    buf[bytesRead] = '\0'; // 确保 buf 末尾是 null 终止
    printf("Read %zd bytes: %s\n", bytesRead, buf);

    // 关闭文件
    close(fd);
    return 0;
}
```

#### 2. 从管道读取数据（父子进程间通信）

```c
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <fcntl.h>
#include <stdlib.h>

int main() {
    int pipefd[2];
    pid_t pid;
    char buf[128];

    // 创建管道
    if (pipe(pipefd) == -1) {
        perror("pipe");
        return -1;
    }

    // 创建子进程
    pid = fork();
    if (pid == -1) {
        perror("fork");
        return -1;
    }

    if (pid == 0) { // 子进程
        close(pipefd[1]); // 关闭写端

        // 从管道中读取数据
        ssize_t bytesRead = read(pipefd[0], buf, sizeof(buf));
        if (bytesRead == -1) {
            perror("read");
            close(pipefd[0]);
            exit(1);
        }

        buf[bytesRead] = '\0'; // 确保字符串结束符
        printf("Child received: %s\n", buf);
        close(pipefd[0]); // 关闭读端
    } else { // 父进程
        close(pipefd[0]); // 关闭读端

        // 向管道写入数据
        const char *msg = "Hello from parent process";
        write(pipefd[1], msg, strlen(msg) + 1); // 包括 null 终止符
        close(pipefd[1]); // 关闭写端
    }

    return 0;
}
```

### `read()` 常见的行为和注意事项：

1. **阻塞行为**：
   - 对于管道、套接字等文件描述符，`read()` 可能会被阻塞，直到有数据可读。如果没有数据，`read()` 会阻塞当前进程，直到另一个进程写入数据。
   - 对于普通文件，`read()` 会直接返回数据，不会阻塞。

2. **文件结束（EOF）**：
   - 如果文件已经读取完毕，`read()` 会返回 0，表示文件结束。
   - 需要注意，在循环读取文件时，如果 `read()` 返回 0，意味着文件已结束，应停止读取。

3. **部分读取**：
   - `read()` 每次调用读取的字节数可能少于请求的字节数。返回值表示实际读取的字节数。
   - 你可能需要循环调用 `read()`，直到读取完所需的字节数或遇到文件结束。

4. **错误处理**：
   - 如果发生错误，`read()` 返回 `-1`，并设置 `errno`。常见的错误包括文件描述符无效、文件系统错误等。

5. **返回值**：
   - 返回 `0`：表示已读取到文件末尾（EOF）。
   - 返回正数：表示实际读取的字节数。
   - 返回 `-1`：表示发生错误，并通过 `errno` 提供具体的错误信息。

### 常见的 `errno` 错误：
- **EBADF**：提供的文件描述符无效，通常是未打开的文件描述符。
- **EFAULT**：`buf` 参数是一个无效的指针。
- **EINTR**：系统调用被信号中断，可以重新调用 `read()`。
- **EAGAIN / EWOULDBLOCK**：非阻塞模式下，`read()` 无法立即返回数据。

### `read()` 与其他 I/O 系统调用的比较：
- **`read()` vs. `fread()`**：`read()` 是系统调用，直接与内核交互，适用于低级文件操作。`fread()` 是标准库函数，通常用于处理文件流，提供更高级的功能，如缓冲和文件指针管理。
- **`read()` vs. `recv()`**：`recv()` 是专门用于读取网络套接字数据的系统调用，适用于基于协议的通信（例如 TCP/IP）。
