OS = 对象 + API

# 16.1 OS的API
OS对象：进程和地址空间
- 进程管理
- 内存管理
- 文件和设备
访问OS中的对象
- 文件：有“名字”的对象
- 字节流（终端）或字节序列

文件描述符
- 指向OS对象的“指针”，通过指针访问一切
- 对象的访问都需要指针

![[Pasted image 20241105200540.png]]

# Take-away Messages

通过 freestanding 的 shell，我们阐释了 “可以在系统调用上创建整个操作系统应用世界” 的真正含义：操作系统的 API 和应用程序是互相成就、螺旋生长的：有了新的应用需求，就有了新的操作系统功能。而 UNIX 为我们提供了一个非常精简、稳定的接口 (fork, execve, exit, pipe ,...)，纵然有沉重的历史负担，它在今天依然工作得很好

---


`gdb -x` 选项用于告诉 GDB 读取并执行指定文件中的 GDB 命令。

具体来说，`-x` 后面跟着一个文件名，该文件中可以包含一系列的 GDB 命令，这些命令在 GDB 启动时自动执行。它的作用类似于在启动 GDB 时手动输入一系列命令，适用于自动化调试或批量执行调试命令的场景。

### 语法：
```bash
gdb -x <command_file> <program>
```

### 示例：
假设你有一个 `commands.gdb` 文件，其中包含了你想在调试时自动执行的命令：

```bash
# commands.gdb 文件内容
break main
run
backtrace
quit
```

你可以通过以下命令启动 GDB 并执行 `commands.gdb` 中的命令：

```bash
gdb -x commands.gdb my_program
```

这将自动在 GDB 启动时设置断点（`break main`），运行程序（`run`），显示堆栈回溯（`backtrace`），然后退出 GDB（`quit`）。


---

这些命令和设置是用来配置 GDB 调试器的行为，以便在调试时更灵活地控制程序的执行流程，并禁用某些不需要的功能。

让我们逐个解释这些命令的作用：

### 1. 调试设置
```bash
set follow-fork-mode child
```
- **跟随子进程**：当程序创建一个子进程时，GDB 会自动跟踪子进程的执行，而不是父进程。

```bash
set detach-on-fork off
```
- **不分离子进程**：此命令告诉 GDB 在 fork 时不要分离子进程。默认情况下，GDB 会选择是否在 fork 时分离子进程，这个设置强制禁用该行为。

```bash
set follow-exec-mode same
```
- **执行模式相同**：在程序执行（`exec`）时，GDB 会继续跟踪原程序进程。如果为 `same`，GDB 会继续跟踪原进程，而不是切换到新的程序。

```bash
set confirm off
```
- **关闭确认提示**：GDB 在某些操作时（如删除断点、退出等）会要求确认，使用 `set confirm off` 可以关闭这些提示，避免需要手动确认。

```bash
set pagination off
```
- **关闭分页**：默认情况下，GDB 输出过长时会分页显示，使用此命令可以关闭分页，输出会一次性显示完。

```bash
tui disable
```
- **禁用 TUI 模式**：TUI (Text User Interface) 模式提供一个更为图形化的界面来调试程序，禁用此模式可以让调试器使用传统的命令行界面。

### 2. 跳过特定函数
```bash
skip function strlen
skip function strcpy
skip function strchr
skip function print
skip function zalloc
skip function peek
skip function gettoken
```
- **跳过特定函数**：这些命令告诉 GDB 在调试时跳过某些特定的函数，不对它们进行单步调试。对于性能敏感或者不感兴趣的函数，跳过它们可以让调试过程更高效。

### 3. 加载 Python 脚本
```bash
source visualize.py
```
- **加载 Python 脚本**：这会加载并执行指定的 Python 脚本（`visualize.py`）。在 GDB 中，你可以使用 Python 来编写脚本进行自动化任务，比如数据可视化、处理特定调试任务等。

### 4. 设置断点
```bash
break main
break runcmd
```
- **设置断点**：这两个命令会在 `main` 函数和 `runcmd` 函数处设置断点。当程序运行到这些函数时，GDB 会暂停执行，允许你进行调试。

### 5. 启动和执行
```bash
run
```
- **运行程序**：这个命令会启动并运行程序，直到遇到断点或程序结束。

```bash
n
```
- **执行下一行**：这个命令会执行当前行，并暂停在下一行。它适用于你已经设置了断点并想逐行调试时。

### 6. 定义停止钩子
```bash
define hook-stop
    pdump
end
```
- **定义停止钩子**：当程序在 GDB 中停止时（例如遇到断点或发生异常），会自动执行 `pdump` 命令（假设这是你预先定义的命令）。`hook-stop` 允许你在程序暂停时自动执行一些操作，通常用于打印状态、日志或者其他诊断信息。

这些配置结合起来会创建一个高效的调试环境，自动化部分过程并让调试更加流畅。如果你有 `visualize.py` 脚本和 `pdump` 命令，这些设置可以帮助你更好地分析程序行为。