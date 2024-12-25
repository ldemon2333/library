操作系统的工作是
1. 将计算机的资源在多个程序间共享，并且给程序提供一系列比硬件本身更有用的服务。
2. 管理并抽象底层硬件，比如，一个文件处理软件（Word）不用去关心自己使用的是何种硬盘。
3. 多路复用硬件，使得多个程序可以(至少看起来是)同时运行的。
4. 给程序间提供一种受控的交互方式，使得程序之间可以共享数据、共同工作。

操作系统通过接口向用户程序提供服务。设计一个好的接口实际上是很难的。一方面我们希望接口设计得简单和精准，使其易于正确地实现；另一方面，我们可能忍不住想为应用提供一些更加复杂的功能。解决这种矛盾的办法是让接口的设计依赖于少量的 _机制_ （_mechanism_)，而通过这些机制的组合提供强大、通用的功能。

本书通过 xv6 操作系统来阐述操作系统的概念，它提供 Unix 操作系统中的基本接口，同时模仿 Unix 的内部设计。Unix 里机制结合良好的窄接口提供了令人吃惊的通用性。这样的接口设计非常成功，使得包括 BSD，Linux，Mac OS X，Solaris （甚至 Microsoft Windows 在某种程度上）都有类似 Unix 的接口。理解 xv6 是理解这些操作系统的一个良好起点。
![[Pasted image 20241014190936.png]]
如图1.1所示，xv6 使用了传统的**内核**概念 - 一个向其他运行中程序提供服务的特殊程序。每一个运行中程序（称之为**进程**）都拥有包含指令、数据、栈的内存空间。指令实现了程序的运算，数据是用于运算过程的变量，栈管理了程序的过程调用。

当进程需要调用内核服务时，它会调用 system call ，这是操作系统接口中的调用之一。 系统调用进入内核；内核执行服务并返回。因此，一个进程交替执行 user space 和 kernel space 。

内核使用了 CPU 的硬件保护机制来保证用户进程只能访问自己的内存空间。内核拥有实现保护机制所需的硬件权限(hardware privileges)，而用户程序没有这些权限。当一个用户程序进行一次系统调用时，硬件会提升特权级并且开始执行一些内核中预定义的功能。

内核提供的一系列系统调用就是用户程序可见的操作系统接口，xv6 内核提供了 Unix 传统系统调用的一部分，它们是：

![[Pasted image 20241209164612.png]]


这一章剩下的部分将说明 xv6 系统服务的概貌 —— 进程，内存，文件描述符，管道和文件系统，为了描述他们，我们给出了代码和一些讨论。这些系统调用在 shell 上的应用阐述了他们的设计是多么独具匠心。

shell 是一个普通的程序，它读取用户的命令并执行它们。shell 是一个用户程序，而不 是内核的一部分，这一事实说明了系统调用接口的强大功能：shell 没有什么特别的。这也 意味着外壳易于更换；因此，现代 Unix 系统有多种 shell 可供选择，每种都有自己的用户 界面和脚本功能。xv6 shell 是 Unix Bourne shell 本质的简单实现。它的实现可以在以下位置找到 (user/sh.c) 。

# 1.1 进程和内存
xv6 进程由用户空间内存（指令、数据和堆栈）和内核私有的进程状态组成。Xv6 *time-share* 进程：它在等待执行的进程集中透明地切换可用CPU。当进程未执行时，xv6会保存进程的CPU寄存器，并在下次运行进程时回复它们。内核将进程标识符（PID）与每个进程关联。

一个进程可以使用fork系统调用创建一个新进程。fork为新进程提供了父进程内存的精确副本，包括指令和数据。fork在父进程和子进程中均返回。在父进程中，fork返回子进程的PID。在子进程中，fork返回0。

![[Pasted image 20241014222031.png]]
该 exit 系统调用导致调用进程停止执行并释放内存和打开文件等资源。exit 采用整数状态参数，通常 0 表示成功，1 表示失败。这 wait 系统调用返回当前进程已退出（或被杀死）的子进程PID，并将子进程的退出状态复制到传递给 wait 的地址；如果调用者的子进程都没有退出，则 wait 等待其中一个退出。如果调用者没有子进程，wait 立即返回 -1。 如果父进程不关心子进程的退出状态，它可以将 0 地址传递给 wait。

`exec` takes two arguments: the name of the file containing the executable and an array of string arguments. For example:
![[Pasted image 20241209165952.png]]
xv6 shell使用以上四个system call来为用户执行程序。在shell进程的`main`中主循环先通过`getcmd`来从用户获取命令，然后调用`fork`来运行一个和当前shell进程完全相同的子进程。父进程调用`wait`等待子进程`exec`执行完（在`runcmd`中调用`exec`）
![[Pasted image 20241209170317.png]]

Xv6 allocates most user-space memory implicitly: `fork` allocates the memory required for the child's copy of the parent's memory, and `exec` allocates enough memory to hold the executable file. A process that needs more memory at run-time (perhaps for `malloc`) can call `sbrk(n)` to grow its data memory by `n` bytes; `sbrk` returns the location of the new memory.

# 1.2 I/O and File descriptors
每个进程都拥有自己独立的文件描述符列表，其中0是标准输入，1是标准输出，2是标准错误。shell将保证总是有3个文件描述符是可用的

The shell ensures that it always has three file descriptors open, which are by default fd for the console.
![[Pasted image 20241209171157.png]]

`read`和`write`：形式`int write(int fd, char *buf, int n)`和`int read(int fd, char *bf, int n)`。从/向文件描述符`fd`读/写n字节`bf`的内容，返回值是成功读取/写入的字节数。每个文件描述符有一个offset，`read`会从这个offset开始读取内容，读完n个字节之后将这个offset后移n个字节，下一个`read`将从新的offset开始读取字节。`write`也有类似的offset。

父进程的fd table将不会被子进程fd table的变化影响，但是文件中的offset将被共享。
![[Pasted image 20241209171432.png]]

The `close` system call releases a file descriptor, making it free for reuse by a future `open`, `pipe`, or `dup` system call. A newly allocated fd is always the lowest-numbered unused descriptor of the current process.

Fd and `fork` interact to make I/O redirection easy to implement. `fork` copies the parent's file descriptor table along with its memory, so that the child starts exactly the same open files as the parent. The system call `exec` replaces the calling process's memory but preserves its file table. This behavior allows the shell to implement I/O redirection by forking reopening chosen fd in the child, and then calling `exec` to run the new program. 

`close`。形式是`int close(int fd)`，将打开的文件`fd`释放，使该文件描述符可以被后面的`open`、`pipe`等其他system call使用。

使用`close`来修改file descriptor table能够实现I/O重定向

![[Pasted image 20241209172241.png]]

`O_TRUNC` to truncate the file to zero length.

Now it should be clear why it is helpful that `fork` and `exec` are separate calls: between the two, the shell has a chance to redirect the child's I/O without disturbing the I/O setup of the main shell.

Although `fork` copies the fd table, each underlying file offset is shared between parent and child. Consider this example:
![[Pasted image 20241209173527.png]]
This behavior helps produce sequential output from sequences of shell commands, like `(echo hello;echo world) > output.txt`.

## dup
`int dup(int fd)`，复制一个新的`fd`指向的I/O对象，返回这个新fd值，两个I/O对象(文件)的offset相同。返回的文件描述符**不一定相同**，而是一个新的、未使用的最小的文件描述符。
![[Pasted image 20241209173819.png]]
除了`dup`和`fork`之外，其他方式**不能**使两个I/O对象的offset相同，比如同时`open`相同的文件

`dup` allows shells to implement commands like this: `ls existing-file non-existing-file > tmp1 2>&1`. The `2>&1` tells the shell to give the command a file descriptor 2 that is a duplicate of descriptor 1. Both the name of the existing file and the error message for the non-existing file will show up in the file `tmp1`. 

# 1.3 Pipes
_pipe_：管道，暴露给进程的一对文件描述符，一个文件描述符用来读，另一个文件描述符用来写，将数据从管道的一端写入，将使其能够被从管道的另一端读出

`pipe`是一个system call，形式为`int pipe(int p[])`，`p[0]`为读取的文件描述符，`p[1]`为写入的文件描述符
![[Pasted image 20241209195129.png]]
The program calls `pipe`, which creates a new pipe and records the read and write file descriptors in the array `p`. After `fork`, both parent and child have fd referring to the pipe. The child calls `close` and `dup` to make fd 0 refer to the read end of the pipe, closes the fd in `p`, and calls `exec` to run `wc`. When `wc` reads from its standard input, it reads from the pipe. The parent closes the read side of the pipe, writes to the pipe, and then closes the write side.

==If no date is available, a `read` on a pipe waits for either data to be written or for all fd referring to the write end to be close==; in the latter case, `read` will return 0, just as if the end of a data file had been reached. 事实上，read 会一直阻塞直到不可能有新数据到达，这是子进程在执行上面的 wc 之前关闭管道写入端非常重要的一个原因：如果 wc 的一个文件描述符引用了管道的写入端，则 wc 永远不会看到文件结束。

先关 pipe 的写端，这样 pipe 的读端就不会阻塞了，然后执行 wc；
```
close(p[1]);
exec("/bin/wc", argv);
```

注意这里关闭p[1]非常重要，因为如果不关闭p[1]，管道的读取端会一直等待读取，wc就永远也无法等到EOF。

The xv6 shell implements pipelines such as `grep fork sh.c | wc -l` in a manner like this. The child process creates a pipe to connect the left end of the pipeline with the right end. Then it calls `fork` and `runcmd` for the left end of the pipeline and `fork` and `runcmd` for the right end, and waits for both to finish. 
![[Pasted image 20241209201003.png]]
![[Pasted image 20241209201625.png]]

在这种情况下，管道至少比临时文件有四个优点。
- 首先，管道会自动清理自身；使用文件重定向时，shell 必须小心地在完成后删除 /tmp/xyz。
- 其次，管道可以传递任意长的数据流，而文件重定向需要磁盘上有足够的可用空间来存储所有数据。
- 第三，管道允许并行执行管道阶段，而文件方法要求第一个程序在第二个程序开始之前完成。
- 第四，如果您正在实现进程间通信，管道的阻塞读写比文件的非阻塞语义更有效率。

# 1.4 File system
xv6文件系统包含了文件(byte arrays)和目录(对其他文件和目录的引用)。目录生成了一个树，树从根目录`/`开始。对于不以`/`开头的路径，认为是是相对路径

- `mknod`：创建设备文件，一个设备文件有一个major device 和一个 minor device 用来唯一确定这个设备。当一个进程打开了这个设备文件时，内核会将`read`和`write`的system call重新定向到设备上。
- 一个文件的名称和文件本身是不一样的，文件本身，也叫 _inode_，可以有多个名字，也叫 _link_，每个 link 包括了一个文件名和一个对 inode 的引用。一个inode存储了文件的元数据，包括该文件的类型(file, directory or device)、大小、文件在硬盘中的存储位置以及指向这个inode的link的个数
- `fstat`。一个system call，形式为`int fstat(int fd, struct stat *st)`，将inode中的相关信息存储到`st`中。
- `link`。一个system call，将创建一个指向同一个inode的文件名。`unlink`则是将一个文件名从文件系统中移除，只有当指向这个inode的文件名的数量为0时这个inode以及其存储的文件内容才会被从硬盘上移除

注意：Unix提供了许多在**用户层面**的程序来执行文件系统相关的操作，比如`mkdir`、`ln`、`rm`等，而不是将其放在shell或kernel内，这样可以使用户比较方便地在这些程序上进行扩展。但是`cd`是一个例外，它是在shell程序内构建的，因为它必须要改变这个calling shell本身指向的路径位置，如果是一个和shell平行的程序，那么它必须要调用一个子进程，在子进程里起一个新的shell，再进行`cd`，这是不符合常理的。
![[Pasted image 20241209202118.png]]
`mknod` creates a special file that refers to a device. Associated with a device file are the major and minor device numbers (the two arguments to `mknod`), which uniquely identify a kernel device. When a process later opens a device file, the kernel diverts `read` and `write` system calls to the kernel device implementation instead of passing them to the file system.

The `fstat` system call retrieves information from the inode that a file descriptor refers to. It fills in a `struct stat`, defined in `stat.h` as:
![[Pasted image 20241209202624.png]]

The `link` system call creates another file system name referring to the same inode as an existing file. This fragment creates a new file named both `a` and `b`.
![[Pasted image 20241209202813.png]]
Reading from or writing to `a` is the same as reading from or writing to `b`. Each inode is identified by a unique *inode number*. After the code sequence above, it is possible to determine that `a` and `b` refer to the same underlying contents by inspecting the result of `fstat`: both will return the same inode number (`ino`), and the `nlink` count will be set to 2.

The `unlink` system call removes a name from the file system. The file's inode and the disk space holding its content are only freed when the file's link count is zero and no file descriptors refer to it.

Furthermore, 
![[Pasted image 20241209203259.png]]
is an idiomatic way to create a temporary inode with no name that will be cleaned up when the process closes `fd` or exits.

Unix provides file utilities callable from the shell as user-level programs, for example `mkdir`, `ln`, and `rm`. This design allows anyone to extend the command-line interface by adding new user-level programs. 事后看来，这个计划似乎很明显，但在 Unix 时代设计的其他系统经常将此类命令内置到 shell 中（并将 shell 内置到内核中）。

One exception is `cd`, which is built into the shell. `cd` must change the current working directory of the shell itself.

