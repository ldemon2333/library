`ftruncate` 是一个系统调用，用于调整文件的大小。它可以用来将文件的大小截断为指定的大小。如果文件的当前大小大于指定的大小，那么文件内容会被截断；如果文件的大小小于指定的大小，则文件会被填充零，直到达到指定的大小。

### 函数原型：

```c
#include <unistd.h>

int ftruncate(int fd, off_t length);
```

- `fd`: 文件描述符，必须指向一个已经打开的文件。
- `length`: 新的文件大小（字节数）。文件将被调整到这个大小。
- 返回值：成功时返回 `0`，失败时返回 `-1` 并设置 `errno`。

### 参数说明：

- **文件描述符（fd）**：文件描述符通常是通过 `open` 或其他文件操作系统调用（如 `creat`）返回的。
- **长度（length）**：这是文件的新大小，单位为字节。如果文件的当前大小大于这个长度，文件会被截断。如果小于该长度，文件会被填充零字节，直到达到指定的长度。

### 使用场景：

- **截断文件**：如果你只需要文件的前 `n` 字节而不关心其他内容，可以使用 `ftruncate` 截断文件。
- **增加文件大小**：在某些情况下，程序可能需要扩展文件的大小（如创建文件的某些预留空间），`ftruncate` 允许文件扩展并填充零字节。
- **数据清理**：将文件截断为零字节大小可以清空文件内容。

### 示例：使用 `ftruncate`

以下是一个简单的例子，演示如何使用 `ftruncate` 调整文件大小：

```c
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>

int main() {
    // 打开文件（如果不存在则创建）
    int fd = open("example.txt", O_RDWR | O_CREAT, S_IRUSR | S_IWUSR);
    if (fd == -1) {
        perror("open");
        return 1;
    }

    // 使用 ftruncate 截断文件
    off_t new_size = 100;  // 目标文件大小为 100 字节
    if (ftruncate(fd, new_size) == -1) {
        perror("ftruncate");
        close(fd);
        return 1;
    }

    printf("File successfully truncated to %ld bytes.\n", new_size);

    // 关闭文件描述符
    close(fd);
    return 0;
}
```

### 解释：

1. **打开文件**：我们使用 `open` 以读写模式打开文件 `example.txt`。如果文件不存在，`O_CREAT` 标志会创建一个新文件。`S_IRUSR | S_IWUSR` 设置文件权限为用户可读可写。
2. **截断文件**：调用 `ftruncate` 将文件的大小设置为 `100` 字节。文件内容会被截断为前 100 字节，超出部分会被删除。
3. **错误处理**：如果 `ftruncate` 失败，它将返回 `-1`，并设置 `errno`。通过 `perror` 打印错误信息。
4. **关闭文件**：最后，使用 `close` 关闭文件描述符。

### 注意事项：

1. **文件偏移**：调用 `ftruncate` 时，文件的偏移量并不会被改变。如果你在调用 `ftruncate` 后继续读取或写入文件，文件描述符的当前偏移量会影响操作的起始位置。
    
2. **文件内容填充**：如果新文件大小大于现有文件的大小，文件会被填充零字节，直到达到新大小。
    
3. **文件类型要求**：`ftruncate` 只适用于普通文件，对于目录文件或符号链接等特殊文件类型，调用 `ftruncate` 会返回错误。
    
4. **文件系统支持**：某些文件系统可能不支持直接扩展文件的大小，尽管大多数现代文件系统都支持此操作。
    

### 通过 `ftruncate` 扩展文件大小

```c
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>

int main() {
    int fd = open("example.txt", O_RDWR | O_CREAT, S_IRUSR | S_IWUSR);
    if (fd == -1) {
        perror("open");
        return 1;
    }

    off_t new_size = 1024;  // 扩展文件到 1024 字节
    if (ftruncate(fd, new_size) == -1) {
        perror("ftruncate");
        close(fd);
        return 1;
    }

    printf("File expanded to %ld bytes.\n", new_size);

    close(fd);
    return 0;
}
```

在这个例子中，我们将文件扩展到 1024 字节。如果文件的原始大小小于 1024 字节，文件将被填充零字节。如果文件已经足够大，则不会有变化。

### 错误处理和 `errno`

`ftruncate` 可能失败，返回 `-1` 并设置 `errno`。常见的错误包括：

- **`EBADF`**：文件描述符 `fd` 不是有效的打开文件描述符。
- **`EINTR`**：调用被信号中断。
- **`EINVAL`**：`length` 参数无效。
- **`ENOSPC`**：磁盘空间不足，无法扩展文件。

你可以通过 `perror` 或 `strerror` 函数获取错误信息并进行处理。

### 总结：

`ftruncate` 是一个非常有用的系统调用，特别适用于调整文件大小，无论是截断文件还是扩展文件。它使得程序能够动态控制文件的大小，而不需要重写整个文件或使用其他复杂的方式。


`truncate` 是一个系统调用，用于调整文件的大小，类似于 `ftruncate`。它可以用来截断文件（将文件内容减少到指定大小），或者将文件扩展为更大的大小（如果指定的大小大于当前文件大小）。与 `ftruncate` 不同的是，`truncate` 是通过文件路径来操作文件，而 `ftruncate` 是通过文件描述符。

### 函数原型：

```c
#include <unistd.h>

int truncate(const char *path, off_t length);
```

- `path`：要调整大小的文件的路径。
- `length`：新的文件大小，以字节为单位。如果文件的当前大小大于这个值，文件会被截断；如果小于这个值，文件将被扩展并用零填充。
- 返回值：成功时返回 `0`，失败时返回 `-1`，并设置 `errno`。

### 参数说明：

- **`path`**：要调整大小的文件的路径。
- **`length`**：目标文件的大小。它将是文件的新长度。文件的实际内容会被截断或扩展至此大小。
    - 如果文件的当前大小大于 `length`，文件的尾部将被删除。
    - 如果文件的当前大小小于 `length`，文件会被扩展，扩展部分会填充零字节，直到文件达到指定大小。

### 使用场景：

- **截断文件**：如果你想删除文件的部分内容，`truncate` 是一个简单的方法。
- **扩展文件**：有时你可能需要为一个文件预留一定的空间（例如，某些日志文件或数据文件），`truncate` 可以扩展文件并填充零字节。
- **清空文件**：将文件大小设置为零字节，实际上会清空文件内容。

### 示例：使用 `truncate`

以下是一个简单的例子，演示如何使用 `truncate` 调整文件的大小：

```c
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>

int main() {
    // 指定要操作的文件路径
    const char *filename = "example.txt";

    // 使用 truncate 将文件大小调整为 100 字节
    off_t new_size = 100;
    if (truncate(filename, new_size) == -1) {
        perror("truncate");
        return 1;
    }

    printf("File successfully truncated to %ld bytes.\n", new_size);

    return 0;
}
```

