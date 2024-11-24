# DESCRIPTION
Pipes and FIFOs (also known as named pipes) provided a unidirectional interprocess communication channel. A pipe has a read end and a write end.

A pipe is created using [[pipe 2]], which creates a new pipe and returns two file descriptors.

A FIFO has a name within the filesystem (created using [[mkfifo 3]]), and is opened using [[open 2]]. Any process may open a FIFO, assuming the file permissions allow it

## I/O on pipes and FIFOs
The only difference between pipes and FIFOs is the manner in which they are created and opened. 完成这些任务后，管道和 FIFO 上的 I/O 具有完全相同的语义。

If a process attempts to read from an empty pipe, then [[read 2]] will block until data is available. If a process attempts to write to a full pipe, then [[write 2]] blocks until sufficient data has been read from the pipe to allow the write to complete.

The communication channel provided by a pipe is a byte stream: there is no concept of message boundaries.

## Pipe capacity
A pipe has a limited capacity. If the pipe is full, then a [[write 2]] will block or fail, depending on whether the O_NONBLOCK flag is set.


---

FIFO（**先进先出**）是一种命名管道，与匿名管道类似，但是 FIFO 可以在文件系统中创建，并且可以由无关进程进行访问。FIFO 允许两个或多个进程进行通信，并且具有双向传输的能力，尽管通常还是单向的。

### 创建和使用 FIFO 的例子

1. **创建 FIFO**：
   使用 `mkfifo()` 或 `mknod()` 系统调用可以创建一个 FIFO 文件。该文件在文件系统中存在，就像常规文件一样，但它表示的是一个管道。

2. **父进程与子进程通过 FIFO 通信**：
   假设我们有两个进程：父进程将数据写入 FIFO 文件，子进程从 FIFO 文件读取数据。

### 示例代码：

#### 1. 创建 FIFO 文件并写入数据（父进程）

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <fcntl.h>

int main() {
    const char *fifo_path = "/tmp/myfifo";  // FIFO 文件路径
    char write_msg[] = "Hello from parent process";

    // 创建 FIFO 文件（如果它不存在）
    if (mkfifo(fifo_path, 0666) == -1) {
        perror("mkfifo");
        exit(EXIT_FAILURE);
    }

    // 打开 FIFO 文件进行写操作
    int fifo_fd = open(fifo_path, O_WRONLY);
    if (fifo_fd == -1) {
        perror("open");
        exit(EXIT_FAILURE);
    }

    // 向 FIFO 写入数据
    write(fifo_fd, write_msg, strlen(write_msg) + 1);
    printf("Parent wrote: %s\n", write_msg);

    // 关闭 FIFO
    close(fifo_fd);

    return 0;
}
```

#### 2. 从 FIFO 读取数据（子进程）

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <fcntl.h>

int main() {
    const char *fifo_path = "/tmp/myfifo";  // FIFO 文件路径
    char read_msg[100];

    // 打开 FIFO 文件进行读操作
    int fifo_fd = open(fifo_path, O_RDONLY);
    if (fifo_fd == -1) {
        perror("open");
        exit(EXIT_FAILURE);
    }

    // 从 FIFO 中读取数据
    read(fifo_fd, read_msg, sizeof(read_msg));
    printf("Child read: %s\n", read_msg);

    // 关闭 FIFO
    close(fifo_fd);

    return 0;
}
```

#### 3. 测试：

1. **步骤 1**：首先运行子进程程序。它将阻塞在 `open(fifo_path, O_RDONLY)` 直到父进程写入数据。
2. **步骤 2**：然后运行父进程程序。它将向 FIFO 写入数据并立即退出。
3. 子进程接着会读取数据并打印出来。

### 解释：
- `mkfifo(fifo_path, 0666)`：创建一个 FIFO 文件，权限为 0666，表示所有用户都可以读写这个 FIFO 文件。
- `open(fifo_path, O_WRONLY)`：父进程以写模式打开 FIFO 文件。
- `open(fifo_path, O_RDONLY)`：子进程以读模式打开 FIFO 文件。
- `write()`：父进程将数据写入 FIFO。
- `read()`：子进程从 FIFO 读取数据。
- FIFO 是单向的，父进程和子进程的通信是通过写入和读取 FIFO 文件实现的。

### 注意事项：
1. FIFO 是阻塞的。如果没有进程打开 FIFO 进行读取，写操作会阻塞；反之，如果没有进程打开 FIFO 进行写入，读操作会阻塞。
2. FIFO 是在文件系统中可见的，你可以像其他文件一样对 FIFO 文件进行操作（例如，查看、删除）。
3. 多个进程可以打开相同的 FIFO 文件进行通信。

### FIFO 与匿名管道的区别：
- **FIFO** 是命名的，可以在文件系统中查看和管理。
- **匿名管道** 没有文件名，只能在相关的父子进程之间通信。

### 使用场景：
FIFO 适合那些需要不同进程通过一个共享文件进行通信的场景，尤其是进程间没有父子关系时，或者需要在不同时间进行通信时。