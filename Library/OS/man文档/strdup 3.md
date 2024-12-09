`strdup` 是一个标准 C 函数，它的作用是创建一个新的字符串副本。具体来说，`strdup` 会分配足够的内存来存储输入字符串的内容，并将该内容复制到新分配的内存区域中。返回的是一个指向新字符串的指针。

### 函数原型：

```c
char *strdup(const char *s);
```

### 参数：

- `s`: 要复制的源字符串。

### 返回值：

- 如果成功，`strdup` 返回一个指向新分配内存区域的指针，其中包含了源字符串的副本。
- 如果内存分配失败，返回 `NULL`。

### 示例代码：

```c
#include <stdio.h>
#include <string.h>

int main() {
    const char *original = "Hello, world!";
    char *copy = strdup(original);

    if (copy != NULL) {
        printf("Original: %s\n", original);
        printf("Copy: %s\n", copy);
        free(copy);  // 不要忘记释放分配的内存
    } else {
        printf("Memory allocation failed.\n");
    }

    return 0;
}
```

### 重要事项：

1. `strdup` 返回的是一个新的指针，指向新分配的内存区域，因此调用者必须在不再需要该副本时使用 `free` 释放内存。
2. 如果传入的字符串 `s` 为 `NULL`，`strdup` 的行为是未定义的，但通常会返回 `NULL`。
3. `strdup` 并不是 C 标准库中的一部分（它是 POSIX 标准的一部分），因此在某些环境下可能无法使用。如果你在某些不支持 `strdup` 的平台上工作，可以手动实现类似的功能，例如：
    
    ```c
    char *my_strdup(const char *s) {
        size_t len = strlen(s) + 1;
        char *copy = (char *)malloc(len);
        if (copy != NULL) {
            memcpy(copy, s, len);
        }
        return copy;
    }
    ```
    

# Description
The `strdup()` function returns a pointer to a new string which is a duplicate of the string s. Memory for the new string is obtained with [[malloc 3]], and can be freed with [[free 3]].
