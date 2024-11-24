# ptrace 系统调用
GDB 实现的核心技术是pstrace()系统调用

# 简易的GDB
实现一个有如下功能的 GDB：
- 可以对一个可执行程序进行调试
- 可以在调试程序时，设置断点
- 可以在调试程序时，打印程序的信息

## 1. 调试可执行文件
我们使用 GDB 调试程序时，一般使用 GDB 直接加载程序的可执行文件，如下命令：

```text
$ gdb ./example
```

上面命令的执行过程如下：
- 首先，GDB 调用 `fork()` 系统调用创建一个新的子进程。
- 然后，子进程会调用 `exec()` 系统调用加载程序的可执行文件到内存。
- 接着，子进程便进入停止状态（停止运行），并且等待 GDB 主进程发送调试命令。

流程如下图所示：
![[Pasted image 20241022133532.png]]
## 第一步：创建被调试子程序
调试程序一般分为 `被调试进程` 与 `调试进程`。

- `被调试进程`：就是需要被调试的进程。
- `调试进程`：主要用于向 被调试进程 发送调试命令。

实现代码如下：
```text
int main(int argc, char** argv)
{
    pid_t child_pid;
 
    if (argc < 2) {
        fprintf(stderr, "Expected a program name as argument\n");
        return -1;
    }
 
    child_pid = fork();
    
    if (child_pid == 0) {               // 1) 子进程：被调试进程
        load_executable_file(argv[1]);  // 加载可执行文件
    } else if (child_pid > 0) {         // 2) 父进程：调试进程
        send_debug_command(child_pid);  // 发送调试命令
    } else {
        perror("fork");
        return -1;
    }
 
    return 0;
}
```
上面的代码执行流程如下：

- 主进程首先调用 `fork()` 系统调用创建一个子进程。
- 然后子进程会调用 `load_executable_file()` 函数加载要进行调试的程序，并且等待主进程发送调试命令。
- 最后主进程会调用 `send_debug_command()` 向被调试进程（子进程）发送调试命令。

所以，接下来我们主要介绍 `load_executable_file()` 和 `send_debug_command()` 这两个函数的实现过程。

## 第二部：加载被调试程序
前面我们说过，子进程主要用于加载被调试的程序，并且等待调试进程（主进程）发送调试命令，现在我们来分析下 `load_executable_file()` 函数的实现：

```text
void load_executable_file(const char *target_file)
{
    /* 1) 运行跟踪(debug)当前进程 */
    ptrace(PTRACE_TRACEME, 0, 0, 0);
 
    /* 2) 加载并且执行被调试的程序可执行文件 */
    execl(target_file, target_file, 0);
}
```
`load_executable_file()` 函数的实现很简单，主要执行流程如下：

- 调用 `ptrace(PTRACE_TRACEME...)` 系统调用告知内核，当前进程可以被进行跟踪，也就是可以被调试。
- 调用 `execl()` 系统调用加载并且执行被调试的程序可执行文件。

首先，我们来看看 `ptrace()` 系统调用的原型定义：
```text
long ptrace(long request,  pid_t pid, void *addr,  void *data);
```

下面我们对其各个参数进行说明：

- `request`：向进程发送的调试命令，可以发送的命令很多。比如上面代码的 `PTRACE_TRACEME` 命令定义为 0，表示能够对进程进行调试。
- `pid`：指定要对哪个进程发送调试命令的进程ID。
- `addr`：如果要读取或者修改进程某个内存地址的内容，就可以通过这个参数指定。
- `data`：如果要修改进程某个地址的内容，要修改的值可以通过这个参数指定，配合 `addr` 参数使用。
所以，代码：

```text
ptrace(PTRACE_TRACEME, 0, 0, 0);
```

的作用就是告知内核，当前进程能够被跟踪（调试）。

接着，当调用 `execl()` 系统调用加载并且执行被调试的程序时，内核会把当前被调试的进程挂起（把运行状态设置为停止状态），等待主进程发送调试命令。

当进程的运行状态被设置为停止状态时，内核会停止对此进程进行调度，除非有其他进程把此进程的运行状态改为可运行状态。

## 第三步：向被调试进程发送调试命令
我们来到最重要的一步了，就是要向被调试的进程发送调试命令。

用过 `GDB` 调试程序的同学都非常熟悉，我们可以向被调试的进程发送 `单步调试`、`打印当前堆栈信息`、`查看某个变量的值` 和 `设置断点` 等操作。

这些命令都可以通过 `ptrace()` 系统调用发送，下面我们介绍一下怎么使用 `ptrace()` 系统调用来对被调试进程进行调试操作。

```text
void send_debug_command(pid_t debug_pid)
{
    int status;
    int counter = 0;
    struct user_regs_struct regs;
    unsigned long long instr;

    printf("Tiny debugger started...\n");
 
    /* 1) 等待被调试进程(子进程)发送信号 */
    wait(&status);
 
    while (WIFSTOPPED(status)) {
        counter++;

        /* 2) 获取当前寄存器信息 */
        ptrace(PTRACE_GETREGS, debug_pid, 0, &regs);

        /* 3) 获取 EIP 寄存器指向的内存地址的值 */
        instr = ptrace(PTRACE_PEEKTEXT, debug_pid, regs.rip, 0);

        /* 打印当前执行中的指令信息 */
        printf("[%u.  EIP = 0x%08llx.  instr = 0x%08llx\n",
               counter, regs.rip, instr);

        /* 4) 将被调试进程设置为单步调试，并且唤醒被调试进程 */
        ptrace(PTRACE_SINGLESTEP, debug_pid, 0, 0);
 
        /* 5) 等待被调试进程(子进程)发送信号 */
        wait(&status);
    }
 
    printf("Tiny debugger exited...\n");
}
```
`send_debug_command()` 函数的实现有点小复杂，我们来分析下这个函数的主要执行流程吧。

- 1. 当被调试进程被内核挂起时，内核会向其父进程发送一个 `SIGCHLD` 信号，父进程可以通过调用 `wait()` 系统调用来捕获这个信息。
- 2. 然后我们在一个循环内，跟踪进程执行指令的过程。
- 3. 通过调用 `ptrace(PTRACE_GETREGS...)` 来获取当前进程所有寄存器的值。
- 4. 通过调用 `ptrace(PTRACE_PEEKTEXT...)` 来获取某个内存地址的值。
- 5. 通过调用 `ptrace(PTRACE_SINGLESTEP...)` 将被调试进程设置为单步调试模式，这样当被调试进程每执行一条指令，都会进入停止状态。

整个调试流程可以归纳为以下的图片：

![[Pasted image 20241022135304.png]]