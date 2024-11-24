# DESCRIPTION
The open() system call opens the file specified by pathname. If the specified file does not exist, it may optionally (if O_CREAT is specified in flags) be created by open().

The return value of open() is a file descriptor, a small, nonnegative integer that is used in subsequent system calls ([[read 2]], [[write 2]], [[lseek 2]], [[fcntl 2]], etc.) to refer to the open file. The file descriptor returned by a successful call will be the lowest-numbered file descriptor not currently open for the process.

By default, the new file descriptor is set to remain open across an [[execve 2]]. The file offset is set to the begining of the file.

A call to open() creates a new open file description, an entry in the systemwide table of open files. The open file description records the file offset and the file status flags (see below). A file descriptor is a reference to an open file description; this reference is unaffected if pathname is subsequently removed or modified to refer to a different file.

The argument flags must include one of the following access modes: O_RDONLY, O_WRONLY, or O_RDWR.

# NOTES
Under Linux, the O_NONBLOCK flag is sometimes used in cases where one wants to open but does not necessarily have the intention to read or write. For example, this may be used to open a device in order to get a file descriptor for use with [[ioctl 2]].



---

`open()` 系统调用是 Linux 和类 Unix 操作系统中用于打开文件或设备的核心操作之一。它返回一个文件描述符，通过该文件描述符可以进行后续的文件读写操作。`open()` 是文件 I/O 操作的起点，用于为进程提供对文件的访问。

### `open()` 系统调用的原型

```c
int open(const char *pathname, int flags, mode_t mode);
```

#### 参数：
- **`pathname`**：要打开的文件的路径，可以是相对路径或绝对路径。
- **`flags`**：用于指定打开文件的方式。它由一组标志常量（`O_RDONLY`, `O_WRONLY`, `O_RDWR` 等）组成，并且可以按位或（`|`）组合。
- **`mode`**：如果要创建新文件或修改现有文件的权限，则需要指定文件的权限。此参数只在文件创建时有效（即当 `flags` 包含 `O_CREAT` 时）。它通常由八进制数表示，如 `0644`。

#### 返回值：
- **成功时**，返回一个非负整数，即文件描述符（`fd`），该文件描述符可以用于后续的读写操作。
- **失败时**，返回 `-1`，并设置 `errno` 变量来指示错误原因。

### `flags` 参数常见值：
- **`O_RDONLY`**：只读模式打开文件。
- **`O_WRONLY`**：只写模式打开文件。
- **`O_RDWR`**：读写模式打开文件。
- **`O_CREAT`**：如果文件不存在，则创建该文件。
- **`O_TRUNC`**：如果文件已存在，并且以写模式打开，则清空文件内容。
- **`O_APPEND`**：将写操作定位到文件的末尾，每次写入数据时都会追加到文件的末尾。
- **`O_EXCL`**：如果使用 `O_CREAT` 并且文件已存在，则 `open()` 调用失败。
- **`O_NONBLOCK`**：非阻塞模式打开文件。用于设备和管道等。
- **`O_SYNC`**：同步文件操作，要求每次写入都同步到磁盘。

### `mode` 参数：
当使用 `O_CREAT` 时，`mode` 参数指定新文件的权限。例如：
- **`0644`**：表示文件所有者有读写权限，其他用户有读取权限。
- **`0755`**：表示文件所有者有读写执行权限，其他用户有读取和执行权限。

### `open()` 示例代码：

#### 1. 打开文件进行读写操作

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

int main() {
    int fd;
    const char *filename = "example.txt";

    // 打开文件以读写模式
    fd = open(filename, O_RDWR);
    if (fd == -1) {
        perror("open");
        return -1;
    }

    // 进行文件操作...

    // 关闭文件
    close(fd);
    return 0;
}
```

#### 2. 创建新文件并写入内容

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

int main() {
    int fd;
    const char *filename = "newfile.txt";
    const char *data = "Hello, this is new data!";

    // 创建新文件并写入数据（文件权限 0644）
    fd = open(filename, O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd == -1) {
        perror("open");
        return -1;
    }

    // 向文件写入数据
    ssize_t bytesWritten = write(fd, data, strlen(data));
    if (bytesWritten == -1) {
        perror("write");
        close(fd);
        return -1;
    }

    printf("Written %zd bytes to %s\n", bytesWritten, filename);

    // 关闭文件
    close(fd);
    return 0;
}
```

#### 3. 以追加模式打开文件

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

int main() {
    int fd;
    const char *filename = "example.txt";
    const char *appendData = "\nThis is additional content.";

    // 打开文件进行追加（O_APPEND）
    fd = open(filename, O_WRONLY | O_APPEND);
    if (fd == -1) {
        perror("open");
        return -1;
    }

    // 向文件末尾写入数据
    ssize_t bytesWritten = write(fd, appendData, strlen(appendData));
    if (bytesWritten == -1) {
        perror("write");
        close(fd);
        return -1;
    }

    printf("Appended %zd bytes to %s\n", bytesWritten, filename);

    // 关闭文件
    close(fd);
    return 0;
}
```

### `open()` 的错误处理：

当 `open()` 调用失败时，返回值为 `-1`，并且可以通过 `errno` 变量获取错误代码。常见的错误包括：

- **`EACCES`**：权限不足，无法访问文件。
- **`ENOENT`**：指定的文件不存在（当没有使用 `O_CREAT` 时）。
- **`EEXIST`**：当 `O_CREAT` 和 `O_EXCL` 一起使用时，文件已存在。
- **`EMFILE`**：进程已打开的文件描述符达到系统限制。
- **`ENFILE`**：系统已打开的文件数达到系统限制。
- **`ENOTDIR`**：路径组件不是目录。
- **`EISDIR`**：尝试打开一个目录进行写操作时发生。
- **`EINVAL`**：无效的 `flags` 参数。

### `open()` 和 `openat()` 的区别：

- **`open()`**：传统的打开文件系统调用，路径是相对于当前工作目录的。
- **`openat()`**：允许通过指定目录文件描述符来打开文件，使得路径可以是相对于指定目录的。这对于处理相对路径、符号链接以及避免竞争条件（如 TOCTOU 漏洞）非常有用。

`openat()` 的原型：

```c
int openat(int dirfd, const char *pathname, int flags, mode_t mode);
```

- **`dirfd`**：目录文件描述符，指定了路径的基准目录。如果是 `AT_FDCWD`，则相当于使用当前工作目录。

### 总结：
- `open()` 是文件操作的基础系统调用，用于打开或创建文件。
- 它通过文件描述符与操作系统内核交互，允许进程对文件进行读写操作。
- `open()` 的 `flags` 参数可以组合多个标志，用来控制文件的打开方式（例如：只读、创建新文件、追加写入等）。
- 当使用 `O_CREAT` 时，可以通过 `mode` 参数指定新文件的权限。
- 通过 `errno` 可以检查 `open()` 调用失败时的具体错误。