# NAME
clone, \_\_clone2, clone3 
create a child process

# 介绍
These system calls create a new ("child") process, in a manner similar to [[fork 2]].

By contrast with [[fork 2]], these system calls provide more precise control over what pieces of execution context are shared between the calling process and the child process. For example, using these system calls, the caller can control whether or not the two processes share the virtual address space, the table of file descriptors, and the table of signal handlers. These system calls also allow the new child process to be placed in separate [[namespaces 7]].


`clone` 是 Linux 系统调用之一，用于创建一个新进程或线程。它比传统的 `fork` 更加灵活，允许调用者指定新进程与父进程之间共享的资源。通过 `clone`，可以创建不同粒度的进程分离，比如完全独立的进程或共享地址空间的线程。

---

### **基本语法**
```c
#include <sched.h>
#include <signal.h>

int clone(int (*fn)(void *), void *stack, int flags, void *arg, ...);
```

#### **参数说明**
1. **`fn`**  
   新创建的进程或线程从 `fn` 函数开始执行，`fn` 返回时新进程退出。

2. **`stack`**  
   指向新进程栈底的指针。调用者需要为新进程分配内存空间（例如使用 `malloc` 或 `alloca`），并将地址传递给 `stack`。

3. **`flags`**  
   控制新进程和父进程共享的资源。常用标志包括：
   - **`CLONE_VM`**：共享内存空间。
   - **`CLONE_FS`**：共享文件系统信息（当前目录等）。
   - **`CLONE_FILES`**：共享文件描述符表。
   - **`CLONE_SIGHAND`**：共享信号处理表。
   - **`CLONE_THREAD`**：使新进程成为线程组的一部分。
   - **`CLONE_PARENT`**：使新进程的父进程与调用者的父进程相同。
   - **`CLONE_CHILD_CLEARTID`** 和 **`CLONE_CHILD_SETTID`**：用于线程库，协助内核管理线程 ID。

4. **`arg`**  
   `fn` 的参数。

5. **可选参数**  
   - `parent_tidptr`：新进程的父进程 ID 指针。
   - `child_tidptr`：新进程的子线程 ID 指针。

---

### **简单示例**

#### 创建线程：
```c
#define _GNU_SOURCE
#include <sched.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

#define STACK_SIZE 1024 * 1024  // 1 MB

int thread_fn(void *arg) {
    printf("Hello from thread! Arg: %s\n", (char *)arg);
    return 0;
}

int main() {
    void *stack = malloc(STACK_SIZE);
    if (!stack) {
        perror("malloc");
        exit(EXIT_FAILURE);
    }

    char *message = "Thread message";
    if (clone(thread_fn, stack + STACK_SIZE, CLONE_VM | CLONE_FS | SIGCHLD, message) == -1) {
        perror("clone");
        free(stack);
        exit(EXIT_FAILURE);
    }

    printf("Hello from main!\n");
    sleep(1);  // Wait for thread to finish
    free(stack);
    return 0;
}
```

**说明**：
- 使用了 `CLONE_VM`，子线程与主线程共享内存。
- 使用 `SIGCHLD` 确保子线程正常结束后可以被回收。

---

### **与 `fork` 和 `pthread_create` 的比较**

| 特性                     | `fork`                          | `pthread_create`         | `clone`                         |
|--------------------------|----------------------------------|--------------------------|----------------------------------|
| 地址空间                 | 独立                           | 共享                     | 可控，取决于 `flags`            |
| 性能                     | 比较慢（复制页表）              | 较快                     | 灵活，但使用复杂               |
| API 难度                 | 简单                           | 简单                     | 复杂                           |
| 灵活性                   | 低                             | 中                       | 高                             |

---

### **注意事项**
1. **栈分配**：`clone` 需要调用者分配和管理线程栈。如果栈空间不足，可能导致崩溃。
2. **资源清理**：调用者需要负责清理线程的资源（如释放栈空间）。
3. **线程同步**：`clone` 只提供基本的线程功能，用户需自行实现同步（如 `pthread_mutex` 或原子操作）。
4. **信号处理**：使用 `CLONE_SIGHAND` 时需要注意信号共享的副作用。

`clone` 的强大灵活性使其成为 Linux 内核线程库（如 `pthread`）的基础，但直接使用时需要谨慎处理细节。


在 Linux 系统中，`clone()` 是一个用于创建新进程的低级系统调用。它的功能类似于 `fork()`，但它提供了更多的灵活性，可以通过设置不同的 `flags` 参数来控制新进程的行为。下面是对 `clone()` 系统调用中常用 `flags` 标志的详细解释：

---

### 1. **`CLONE_VM`**

- **作用**: 共享调用进程的虚拟内存空间。
- **解释**: 新进程和父进程共享相同的内存空间，包括堆、栈和代码段。
- **应用场景**: 适用于需要共享内存的多线程环境。
- **注意事项**: 子进程对内存的修改会影响父进程。

---

### 2. **`CLONE_FS`**

- **作用**: 共享文件系统信息。
- **解释**: 新进程和父进程共享文件系统相关的数据结构，如当前工作目录和根目录。
- **应用场景**: 适用于需要共享文件系统上下文的场景。
- **注意事项**: 子进程无法独立更改文件系统的上下文。

---

### 3. **`CLONE_FILES`**

- **作用**: 共享文件描述符表。
- **解释**: 新进程和父进程共享文件描述符表，即打开的文件和套接字。
- **应用场景**: 适用于需要共享文件句柄的场景。
- **注意事项**: 子进程对文件描述符的操作会影响父进程。

---

### 4. **`CLONE_SIGHAND`**

- **作用**: 共享信号处理表。
- **解释**: 新进程和父进程共享信号处理函数和信号屏蔽掩码。
- **应用场景**: 适用于需要共享信号处理机制的场景。
- **注意事项**: 子进程无法独立设置自己的信号处理方式。

---

### 5. **`CLONE_THREAD`**

- **作用**: 将新进程标记为线程。
- **解释**: 新进程被视为父进程的一个线程，而不是独立的进程。
- **应用场景**: 适用于实现用户态线程或轻量级任务。
- **注意事项**: 子进程不会独立显示在进程表中。

---

### 6. **`CLONE_NEWPID`**

- **作用**: 创建新的 PID 命名空间。
- **解释**: 新进程在一个独立的 PID 命名空间中运行，与其他进程隔离。
- **应用场景**: 适用于容器化环境或需要隔离进程 ID 的场景。
- **注意事项**: 子进程的 PID 在其命名空间内独立编号。

---

### 7. **`CLONE_NEWNET`**

- **作用**: 创建新的网络命名空间。
- **解释**: 新进程在一个独立的网络命名空间中运行，拥有独立的网络接口和路由表。
- **应用场景**: 适用于容器化环境或需要网络隔离的场景。
- **注意事项**: 子进程无法访问父进程的网络资源。

---

### 8. **`CLONE_NEWNS`**

- **作用**: 创建新的挂载命名空间。
- **解释**: 新进程在一个独立的挂载命名空间中运行，拥有独立的文件系统挂载点。
- **应用场景**: 适用于容器化环境或需要文件系统隔离的场景。
- **注意事项**: 子进程无法访问父进程的挂载点。

---

### 9. **`CLONE_SYSVSEM`**

- **作用**: 共享 System V 信号量。
- **解释**: 新进程和父进程共享 System V 信号量资源。
- **应用场景**: 适用于需要共享同步机制的场景。
- **注意事项**: 子进程对信号量的操作会影响父进程。

---

### 10. **`SIGCHLD`（与 `clone()` 结合使用）**

- **作用**: 控制子进程终止时是否向父进程发送信号。
- **解释**: 如果未设置 `SIGCHLD`，父进程不会收到子进程终止的通知。
- **应用场景**: 适用于不需要监控子进程状态的场景。
- **注意事项**: 如果父进程需要捕获子进程的终止状态，则必须设置 `SIGCHLD`。

---

### 总结

通过设置不同的 `flags` 标志，`clone()` 系统调用可以灵活地控制新进程的行为，满足不同的需求。常见的组合包括：

- **多线程环境**: 使用 `CLONE_VM` 和 `CLONE_THREAD`。
- **容器化环境**: 使用 `CLONE_NEWPID`、`CLONE_NEWNET` 和 `CLONE_NEWNS`。
- **独立进程**: 不设置任何标志，默认创建独立的子进程。

