`fstat` 是一个用于获取文件状态的系统调用，它用于返回与给定文件描述符关联的文件的元数据。在类 Unix 系统（如 Linux）中，`fstat` 提供了文件的信息，如文件大小、权限、设备信息等。

### 函数原型：

```c
#include <sys/stat.h>
#include <unistd.h>

int fstat(int fd, struct stat *statbuf);
```

- `fd`: 文件描述符，指向一个已经打开的文件。
- `statbuf`: 一个 `struct stat` 类型的指针，系统调用会将文件的状态信息填充到这个结构体中。
- 返回值：成功时返回 0，失败时返回 -1，并设置 `errno`。

### `struct stat` 结构体：

`struct stat` 是一个用于存储文件状态信息的结构体，定义如下：

```c
struct stat {
    dev_t     st_dev;     /* 文件的设备 */
    ino_t     st_ino;     /* 文件的 inode 节点 */
    mode_t    st_mode;    /* 文件的类型和权限 */
    nlink_t   st_nlink;   /* 文件的硬链接数 */
    uid_t     st_uid;     /* 文件所有者的用户 ID */
    gid_t     st_gid;     /* 文件所有者的组 ID */
    dev_t     st_rdev;    /* 如果是设备文件，保存设备类型 */
    off_t     st_size;    /* 文件的大小（字节数） */
    time_t    st_atime;   /* 最后访问时间 */
    time_t    st_mtime;   /* 最后修改时间 */
    time_t    st_ctime;   /* 最后状态变化时间 */
    blksize_t st_blksize; /* 文件的系统 I/O 块大小 */
    blkcnt_t  st_blocks;  /* 文件所占的块数 */
};
```

### 常见的字段说明：

- **st_mode**：文件的类型和权限，通常用来判断文件是常规文件、目录、符号链接等。
- **st_size**：文件的大小，以字节为单位。
- **st_atime、st_mtime、st_ctime**：分别是文件的访问时间、修改时间和状态变化时间。
- **st_uid、st_gid**：文件所有者的用户和组 ID。
- **st_dev** 和 **st_rdev**：设备文件相关的信息。

### 示例：使用 `fstat`

以下是一个示例程序，演示了如何使用 `fstat` 获取文件的元数据：

```c
#include <stdio.h>
#include <sys/stat.h>
#include <unistd.h>

int main() {
    struct stat statbuf;

    // 打开文件
    int fd = open("example.txt", O_RDONLY);
    if (fd == -1) {
        perror("open");
        return 1;
    }

    // 获取文件状态信息
    if (fstat(fd, &statbuf) == -1) {
        perror("fstat");
        close(fd);
        return 1;
    }

    // 打印文件信息
    printf("File Size: %ld bytes\n", statbuf.st_size);
    printf("File Permissions: %o\n", statbuf.st_mode & 0777);  // 打印文件权限
    printf("Number of Hard Links: %ld\n", statbuf.st_nlink);
    printf("Last Access Time: %ld\n", statbuf.st_atime);
    printf("Last Modified Time: %ld\n", statbuf.st_mtime);
    printf("File Owner UID: %d\n", statbuf.st_uid);
    printf("File Owner GID: %d\n", statbuf.st_gid);

    // 关闭文件描述符
    close(fd);
    return 0;
}
```

### 解析：

1. **打开文件**：使用 `open` 函数打开文件，获取文件描述符。
2. **调用 `fstat`**：通过 `fstat` 获取文件的状态信息，并将其存储在 `struct stat` 结构体中。
3. **打印文件信息**：通过打印结构体中的不同字段，展示文件的大小、权限、时间戳等信息。
4. **关闭文件**：最后通过 `close` 关闭文件描述符。

### 使用场景：

- **获取文件信息**：可以用 `fstat` 获取文件的大小、权限、最后访问时间等信息，这在文件管理、文件系统分析等场景中非常有用。
- **文件权限检测**：你可以使用 `fstat` 来检查文件的类型和权限，决定是否执行某些操作。
- **优化 I/O 操作**：通过 `st_blksize`，你可以获知文件的系统 I/O 块大小，这有助于优化文件读取操作。

### `fstat` 与 `stat` 的区别：

- `fstat` 通过文件描述符来获取文件状态信息，而 `stat` 需要文件的路径名。
- 使用 `fstat` 时，文件必须已经打开，而 `stat` 只需要文件路径即可。

### 结论：

`fstat` 是获取已打开文件的元数据的有效方式，能够帮助程序获取文件的详细信息，特别是在文件 I/O 操作、文件系统管理、文件权限控制等方面具有重要作用。如果你有更多具体的用例或问题，欢迎随时询问！


`stat` 是一个用于获取文件或目录的元数据的系统调用，它返回有关指定文件的信息，包括文件大小、权限、文件类型、时间戳等。这与 `fstat` 类似，不同之处在于 `stat` 使用的是文件路径，而 `fstat` 使用的是文件描述符。

### 函数原型：

```c
#include <sys/stat.h>
#include <unistd.h>

int stat(const char *path, struct stat *statbuf);
```

- `path`: 文件或目录的路径。
- `statbuf`: 用于存储文件状态信息的 `struct stat` 结构体指针。
- 返回值：成功时返回 0，失败时返回 -1，并设置 `errno`。


# NAME
stat, fstat, lstat, fstatat - get file status

# 介绍
These functions return information about a file, in the buffer pointed to by _statbuf_. 