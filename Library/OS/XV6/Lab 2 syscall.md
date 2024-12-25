# System call tracing
In this assignment you will add a system call tracing feature that may help you when debugging later labs. You'll create a new `trace` system call that will control tracing. It should take one argument, an integer "mask", whose bits specify which system calls to trace. For example, to trace the fork system call, a program calls `trace(1 << SYS_fork)`, where `SYS_fork` is a syscall number from `kernel/syscall.h`. You have to modify the xv6 kernel to print out a line when each system call is about to return, if the system call's number is set in the mask. The line should contain the process id, the name of the system call and the return value; you don't need to print the system call arguments. The `trace` system call should enable tracing for the process that calls it and any children that it subsequently forks, but should not affect other processes.

在此作业中，您将添加一个系统调用跟踪功能，该功能可能有助于您调试后续的实验。您将创建一个新的跟踪系统调用来控制跟踪。它应该接受一个参数，即一个整数“掩码”，其位指定要跟踪哪些系统调用。例如，要跟踪 fork 系统调用，程序会调用 trace(1 << SYS_fork)，其中 SYS_fork 是 kernel/syscall.h 中的系统调用编号。如果系统调用的编号在掩码中设置，则必须修改 xv6 内核以在每个系统调用即将返回时打印出一行。该行应包含进程 ID、系统调用的名称和返回值；您不需要打印系统调用参数。跟踪系统调用应该启用对调用它的进程及其随后分叉的任何子进程的跟踪，但不应影响其他进程。

Some hints:

- Add `$U/_trace` to UPROGS in Makefile
- Run `make qemu` and you will see that the compiler cannot compile `user/trace.c`, because the user-space stubs for the system call don't exist yet: add a prototype for the system call to `user/user.h`, a stub to `user/usys.pl`, and a syscall number to `kernel/syscall.h`. The Makefile invokes the perl script `user/usys.pl`, which produces `user/usys.S`, the actual system call stubs, which use the RISC-V `ecall` instruction to transition to the kernel. Once you fix the compilation issues, run `trace 32 grep hello README`; it will fail because you haven't implemented the system call in the kernel yet.
- Add a `sys_trace()` function in `kernel/sysproc.c` that implements the new system call by remembering its argument in a new variable in the `proc` structure (see `kernel/proc.h`). The functions to retrieve system call arguments from user space are in `kernel/syscall.c`, and you can see examples of their use in `kernel/sysproc.c`.
- Modify `fork()` (see kernel/proc.c) to copy the trace mask from the parent to the child process.
- Modify the syscall() function in `kernel/syscall.c` to print the trace output. You will need to add an array of syscall names to index into.

## 系统调用过程
![[Pasted image 20241211183715.png]]
所以 `li a7, SYS_fork` 的意思就是，把 `fork` 的调用号赋值到 a7 寄存器内，这样进入内核之后，我们就知道之前调用的是哪个系统调用。

汇编的下一行是 `ecall`。这是一个 risc-v 架构里比较神奇的指令

`ECALL` instruction does an atomic jump to a controlled location (i.e. RISC-V 0x8000 0180)
- Switches the sp to the kernel stack
- Saves the old (user) SP value
- Saves the old (user) PC value (= return address)
- Saves the old privilege mode
- Sets the new privilege mode to 1
- Sets the new PC to the kernel syscall handler

ecall 把我们跳到内核之后，会先进入一个内核的处理函数，`syscall()`。

![[Pasted image 20241211184113.png]]

这个 `syscall` 会根据 a7 寄存器中存的调用号，去调用相应的服务。那如何去通过调用号来得到相应的函数呢？答案就是一个指向函数指针的数组，

这里的 `[SYS_fork] sys_fork` 是 C 语言数组的一个语法，表示以方括号内的值作为元素下标。比如 `int arr[] = {[3] 2333, [6] 6666}` 代表 arr 的下标 3 的元素为 2333，下标 6 的元素为 6666，其他元素填充 0 的数组。（该语法在 C++ 中已不可用）

调用完成之后，系统调用的返回值会在返回用户态时，被赋到 a0 寄存器上，也就是 `p->trapframe->a0 = syscalls[num]();` 这句话的用处。

在 C 语言中，`static uint64 (*syscalls[])(void)` 是一个声明，它涉及数组、函数指针和类型定义的组合。我们可以逐步解析这个声明的含义。

### 解析 `static uint64 (*syscalls[])(void)`：

1. **`static`**:
    
    - 关键字 `static` 表示变量的作用域是局部的，但生命周期是全局的。这意味着该数组只能在声明它的文件中访问，并且它在程序的整个生命周期中保持有效。
    - 对于数组 `syscalls`，`static` 表示它在文件内的作用域是局部的，外部无法直接访问该数组。
2. **`uint64`**:
    
    - `uint64` 是一个数据类型，它通常表示一个 64 位的无符号整数（`unsigned long long` 在 C 中），这表示该函数的返回值类型是 64 位无符号整数。
3. **`(*syscalls[])`**:
    
    - `syscalls` 是一个数组。`[]` 表示这是一个数组，数组的元素是函数指针。函数指针是指向某个函数的指针，函数的类型是 `uint64 (*)(void)`，即返回值类型是 `uint64`，而函数没有参数（即 `void`）。
    - `(*syscalls[])` 表示 `syscalls` 是一个数组，数组中的每个元素都是一个指向返回 `uint64` 且没有参数的函数的指针。
4. **`(void)`**:
    
    - `void` 表示这些函数不接受任何参数。即数组 `syscalls` 中每个元素指向的函数是无参数的。

### 总结：

`static uint64 (*syscalls[])(void)` 的声明表示：

- `syscalls` 是一个静态数组，数组中的每个元素都是一个指向函数的指针。
- 每个函数的返回类型是 `uint64`，且没有参数。

#### 具体示例

例如，假设我们在代码中定义了一些系统调用函数（返回类型是 `uint64`，且无参数），并将它们存储在 `syscalls` 数组中：

```c
#include <stdio.h>

typedef unsigned long long uint64;

// 例子函数，返回一个常量
static uint64 syscall1(void) {
    return 1234567890;
}

// 另一个例子函数
static uint64 syscall2(void) {
    return 9876543210;
}

// 定义 syscalls 数组，包含指向 syscall1 和 syscall2 的指针
static uint64 (*syscalls[])(void) = {syscall1, syscall2};

int main() {
    // 通过 syscalls 数组调用函数
    printf("syscall1: %llu\n", syscalls[0]());  // 调用 syscall1
    printf("syscall2: %llu\n", syscalls[1]());  // 调用 syscall2

    return 0;
}
```

在这个例子中：

- `syscalls` 数组包含两个元素，每个元素都是一个指向返回类型为 `uint64` 且无参数的函数的指针。
- `syscalls[0]()` 调用 `syscall1()`，返回一个值。
- `syscalls[1]()` 调用 `syscall2()`，返回另一个值。

### 关键点

- `syscalls[]` 是一个指向函数的指针数组。
- 每个指针指向一个返回 `uint64` 且没有参数的函数。
- `static` 限制了数组 `syscalls` 只能在当前文件中访问。

## 在各种文件中“注册”系统调用
- 在用户态 `user/user.h` 中申明，使得用户能通过调用这个接口去调用汇编代码，从而进入内核
- 如前文所讲，我们需要使用汇编去实现这个跳转函数。不过，这个汇编是 perl 的脚本自动生成的，所以需要去更改这个脚本（`user/usys.pl`）。
- 到此为止已经完成了在用户态的注册。接下来需要在内核中注册。
- 现在我们需要在 `kernel/syscall.h` 给这个新的调用注册一个调用号，这样才能通过调用号找到函数。
- 然后，就像之前介绍的，内核中的中转函数 `syscall()` 需要通过一个函数指针数组来查找需要调用的函数，所以我们需要去在这个数组中新加一个元素，并且申明一下这个 trace 函数。
- 如前文所讲，像 `extern uint64 sys_trace(void);` 这样的申明是在 `kernel/syscall.c` 中的，而实现在 `kernel/sysproc.c` 中，我们需要到这个文件中随便添加一个实现（具体的实现在下文讲）。

这些函数最后都会调用到一个叫做 `argraw()` 的函数，实现如下，其参数 `n` 代表现在希望读取的是第几个参数：
![[Pasted image 20241211190443.png]]
可以看到其读取了 `trapframe` 中的数据。其实这个 `trapframe` 就是用来给系统调用保存现场的，它记录了发生系统调用时的寄存器状态，以及当前进程内核栈的位置，内核的页表等数据，在完成系统调用后，我们可以根据这里储存的数据，来恢复到调用之前的状态

那为什么我们想要取第几个参数，就返回 `trapframe` 的 a 几呢？虽然我不是很清楚，但大概是因为 risc-v 的函数调用约定

gcc 对于 risc-v 使用的部分函数调用约定有下面几点：
- 返回值（32 位 int）放在 a0 寄存器中
- 参数（32 位 int）从左到右放在 a0、a1……a7 中。如果还有，则从右往左压栈，第 9 个参数在栈顶。

```
user/user.h:		用户态程序调用跳板函数 trace()
user/usys.S:		跳板函数 trace() 使用 CPU 提供的 ecall 指令，调用到内核态
kernel/syscall.c	到达内核态统一系统调用处理函数 syscall()，所有系统调用都会跳到这里来处理。
kernel/syscall.c	syscall() 根据跳板传进来的系统调用编号，查询 syscalls[] 表，找到对应的内核函数并调用。
kernel/sysproc.c	到达 sys_trace() 函数，执行具体内核操作
```

这么繁琐的调用流程的主要目的是实现用户态和内核态的良好隔离。

并且由于内核与用户进程的页表不同，寄存器也不互通，所以参数无法直接通过 C 语言参数的形式传过来，而是需要使用 argaddr、argint、argstr 等系列函数，从进程的 trapframe 中读取用户进程寄存器中的参数。

同时由于页表不同，指针也不能直接互通访问（也就是内核不能直接对用户态传进来的指针进行解引用），而是需要使用 copyin、copyout 方法结合进程的页表，才能顺利找到用户态指针（逻辑地址）对应的物理内存地址。（在本 lab 第二个实验会用到）

# Sysinfo
In this assignment you will add a system call, sysinfo, that collects information about the running system. The system call takes one argument: a pointer to a struct sysinfo (see kernel/sysinfo.h). The kernel should fill out the fields of this struct: the freemem field should be set to the number of bytes of free memory, and the nproc field should be set to the number of processes whose state is not UNUSED. We provide a test program sysinfotest; you pass this assignment if it prints "sysinfotest: OK".


这段代码定义了一个匿名结构体，并将其赋值给变量 `kmem`。具体来说，它定义了一个包含以下两个成员的结构体：

1. **`struct spinlock lock;`**：
    - `lock` 是一个 `spinlock` 类型的成员，通常用于多线程环境中的同步机制。`spinlock` 是一种自旋锁，常用于在多核处理器系统中进行线程同步时避免过多的上下文切换。当一个线程尝试获取一个自旋锁时，如果锁被其他线程持有，它将不断地“自旋”而不是被挂起，直到锁被释放。
2. **`struct run *freelist;`**：
    - `freelist` 是一个指向 `struct run` 类型的指针，通常用于链表结构中的指针，管理空闲内存块或者对象池。`struct run` 可能是一个表示内存块的结构体（在内核中，这样的结构体通常用于管理内存页或者其他内存资源）。

### 这是什么定义？

这是一个**匿名结构体**，意味着没有为这个结构体类型命名，而是直接定义了一个结构体实例。`kmem` 是这个结构体类型的实例。

它的定义相当于：

```c
struct kmem_type {
    struct spinlock lock;
    struct run *freelist;
} kmem;
```

这里的 `kmem` 是类型为 `struct kmem_type` 的一个变量，并且该结构体包含了一个 `spinlock` 类型的成员 `lock` 和一个 `struct run *` 类型的成员 `freelist`。

### 使用这种定义的场景

这种结构体的定义常见于内核编程，特别是在操作系统或内存管理中。以下是一些可能的用例：

1. **内存池管理**：
    
    - `kmem` 可能是一个用于管理内存池的结构体，`lock` 用于保护空闲内存块链表 `freelist`，避免多个线程同时修改它。
2. **多线程同步**：
    
    - `spinlock` 用于确保对 `freelist` 的访问是同步的，防止多个线程/进程同时修改该空闲链表。
3. **动态内存分配**：
    
    - `freelist` 可能是一个指向空闲内存块的链表指针，结构体管理内存的分配和释放。

### 总结

这个定义表示了一个包含 `spinlock` 和 `freelist` 的结构体实例 `kmem`，它常用于内核或低层的内存管理系统中。`freelist` 可能是一个空闲内存块链表，而 `lock` 用于多线程环境中的同步。