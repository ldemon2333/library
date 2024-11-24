# NAME 
lseek - reposition read/write file offset

# SYNOPSIS
```
#include <sys/types.h>
#include <unistd.h>

off_t lseek(int fd, off_t offset, int whence);
```

# DESCRIPTION
lseek() repositions the file offset of the open file description associated with the file descriptor fd to the argument offset according to the directive whence as follows:

lseek() allows the file offset to be set beyond the end of the file (but this does not change the size of the file). If data is later written at this point, subsequent reads of the data in the gap (a "hole") return null bytes ('\0') until data is actually written into the gap.

lseek() 允许将文件偏移量设置为超出文件末尾（但这不会改变文件的大小）。如果稍后在此点写入数据，则后续读取间隙（“空洞”）中的数据将返回空字节（'\0'），直到数据实际写入间隙。

# NOTES
See [[open 2]] for a discussion of the relationship between file descriptors, open file descriptions, and files.

---
`lseek` 是一个系统调用，用于在文件中调整文件指针的位置，使得程序可以随机访问文件的不同位置。它通常用于在已经打开的文件描述符上进行定位操作。

### 原型
```c
#include <unistd.h>
#include <fcntl.h>

off_t lseek(int fd, off_t offset, int whence);
```

### 参数
- `fd`: 文件描述符，表示已经打开的文件。
- `offset`: 相对于 `whence` 参数指定的位置，移动的字节数。
- `whence`: 定位的起始位置，它是下列常量之一：
  - `SEEK_SET`: 从文件的起始位置开始定位，`offset` 是相对于文件开始的位置。
  - `SEEK_CUR`: 从当前文件指针位置开始定位，`offset` 是相对于当前位置的偏移量。
  - `SEEK_END`: 从文件的末尾位置开始定位，`offset` 是相对于文件末尾的偏移量。如果 `offset` 为负数，则表示从文件末尾回退。

### 返回值
- 成功时，返回新的文件指针位置（即文件指针的偏移量，以字节为单位）。
- 失败时，返回 `-1`，并将 `errno` 设置为相应的错误代码。

### 错误码
- `EBADF`: `fd` 不是有效的文件描述符。
- `EINVAL`: `whence` 参数的值无效。
- `ESPIPE`: 文件描述符 `fd` 指向的是一个不能定位的文件类型（例如管道或套接字）。

### 常见用途
1. **随机访问文件**：通过调整文件指针的位置，可以在文件中任意位置进行读取或写入，而不必按顺序读取。
2. **文件大小查询**：可以使用 `lseek` 将文件指针移至文件的末尾，结合 `read` 或 `write` 来获取文件的大小。
3. **修改文件内容**：在某些情况下，可能需要将文件指针移动到特定位置，以便覆盖某部分数据。

### 示例代码
```c
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>

int main() {
    int fd = open("example.txt", O_RDWR);
    if (fd == -1) {
        perror("Error opening file");
        return 1;
    }

    // 移动文件指针到文件的第 10 个字节
    if (lseek(fd, 10, SEEK_SET) == -1) {
        perror("Error seeking in file");
        close(fd);
        return 1;
    }

    // 从当前位置读取数据
    char buffer[10];
    ssize_t bytes_read = read(fd, buffer, sizeof(buffer));
    if (bytes_read == -1) {
        perror("Error reading file");
        close(fd);
        return 1;
    }
    
    printf("Read %zd bytes: %.*s\n", bytes_read, (int)bytes_read, buffer);

    // 将文件指针移到文件末尾，获取文件大小
    off_t file_size = lseek(fd, 0, SEEK_END);
    if (file_size == -1) {
        perror("Error getting file size");
        close(fd);
        return 1;
    }
    printf("File size: %ld bytes\n", file_size);

    close(fd);
    return 0;
}
```

### 解释
- `lseek(fd, 10, SEEK_SET)`：将文件指针移动到文件的第 10 个字节位置。
- `lseek(fd, 0, SEEK_END)`：将文件指针移动到文件的末尾，并返回文件的大小（即文件末尾的位置）。
- `read`：从文件中读取数据。

### 总结
`lseek` 是一个强大的工具，允许我们在文件中随机访问指定位置。通过合理地使用 `lseek`，我们可以高效地定位文件中的任何位置，进行读写操作或查询文件大小。