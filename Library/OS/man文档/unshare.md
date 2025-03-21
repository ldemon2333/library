`unshare` 命令用于在新的命名空间中运行程序，通常用于测试或开发容器化应用。通过 `unshare`，你可以创建一个新的进程，并让它拥有独立的命名空间（例如，进程、网络、挂载等），不受宿主系统影响。这个命令非常适合做系统隔离的测试。

### 基本语法

```bash
unshare [选项] [命令]
```

### 常用选项

- `--mount`: 创建新的挂载命名空间。
- `--uts`: 创建新的 UTS（Unix Time Sharing）命名空间，通常用于隔离主机名和域名。
- `--ipc`: 创建新的 IPC（Inter-Process Communication）命名空间。
- `--net`: 创建新的网络命名空间。
- `--pid`: 创建新的进程命名空间。
- `--user`: 创建新的用户命名空间。
- `--cgroup`: 创建新的 cgroup 命名空间。

### 示例

#### 1. 创建一个新的进程命名空间

这个例子启动一个新的 shell 进程，在一个独立的进程命名空间中运行，当前 shell 将是进程树的根节点。

```bash
unshare --pid --fork bash
```

在这个命令中，`--pid` 选项表示创建一个新的进程命名空间，`--fork` 表示执行 `bash` 命令，启动一个新的 shell。你将在一个新的命名空间中运行，并且在这个命名空间中启动的进程不会出现在宿主机的进程树中。

#### 2. 创建一个新的网络命名空间

如果你想让一个进程拥有独立的网络栈，可以使用 `--net` 选项。这样，进程会有自己独立的 IP 配置、路由表、网络接口等。

```bash
unshare --net bash
```

这个命令会启动一个新的 `bash` 进程，并将其放入一个新的网络命名空间中。你可以在其中运行 `ip a` 来查看网络接口，通常你会看到只有 `lo`（本地回环接口），而不会看到宿主机的网络接口。

#### 3. 创建新的挂载命名空间

使用 `--mount` 选项，可以让新进程有一个独立的文件系统挂载状态。例如，挂载新的文件系统而不影响宿主机的文件系统。

```bash
unshare --mount bash
```

这个命令启动一个新的 shell 进程，并将其放入一个新的挂载命名空间中。你可以在其中执行挂载操作，而不会影响宿主机的文件系统。

#### 4. 创建多个命名空间

你也可以将多个命名空间一起使用。例如，创建一个新的网络、进程和挂载命名空间：

```bash
unshare --net --pid --mount bash
```

这会在一个新的进程命名空间、网络命名空间和挂载命名空间中启动一个新的 `bash`。

### 查看命名空间

你可以通过查看 `/proc/<pid>/ns/` 来查看进程的命名空间，例如：

```bash
ls -l /proc/$$/ns/
```

它会显示当前进程的各种命名空间，例如：

```
lrwxrwxrwx 1 root root 0 Feb 19 10:00 ipc -> ipc:[4026531835]
lrwxrwxrwx 1 root root 0 Feb 19 10:00 mnt -> mnt:[4026531835]
lrwxrwxrwx 1 root root 0 Feb 19 10:00 net -> net:[4026531835]
lrwxrwxrwx 1 root root 0 Feb 19 10:00 pid -> pid:[4026531835]
lrwxrwxrwx 1 root root 0 Feb 19 10:00 uts -> uts:[4026531835]
```

每个命名空间都有一个独立的标识符（如 `ipc:[4026531835]`）。

https://stackoverflow.com/questions/44666700/unshare-pid-bin-bash-fork-cannot-allocate-memory

# unshare --pid /bin/bash -fork cannot allocate memory
I'm experimenting with linux namespaces. Specifically the pid namespace.

I thought I'd test something out with bash but run into this problem:
```bash
unshare -p /bin/bash
bash: fork: Cannot allocate memory
```

Running ls from there gave a core dump. Exit is the only thing possible.

Why is it doing that?

## Answer
After bash start to run, bash will fork several new sub-processes to do somethings. If you run unshare without -f, bash will have the same pid as the current "unshare" process. The current "unshare" process call the unshare systemcall, create a new pid namespace, but the current "unshare" process is not in the new pid namespace. It is the desired behavior of linux kernel: process A creates a new namespace, the process A itself won't be put into the new namespace, only the sub-processes of process A will be put into the new namespace. So when you run:
```
unshare -p /bin/bash
```
The unshare process will exec /bin/bash, and /bin/bash forks several sub-processes, the first sub-process of bash will become PID 1 of the new namespace, and the subprocess will exit after it completes its job. So the PID 1 of the new namespace exits.

The PID 1 process has a special function: it should become all the orphan processes' parent process. If PID 1 process in the root namespace exits, kernel will panic. If PID 1 process in a sub namespace exits, linux kernel will call the disable_pid_allocation function, which will clean the PIDNS_HASH_ADDING flag in that namespace. When linux kernel create a new process, kernel will call alloc_pid function to allocate a PID in a namespace, and if the PIDNS_HASH_ADDING flag is not set, alloc_pid function will return a -ENOMEM error. That's why you got the "Cannot allocate memory" error.

You can resolve this issue by use the '-f' option:

```
unshare -fp /bin/bash
```

If you run unshare with '-f' option, unshare will fork a new process after it create the new pid namespace. And run /bin/bash in the new process. The new process will be the pid 1 of the new pid namespace. Then bash will also fork several sub-processes to do some jobs. As bash itself is the pid 1 of the new pid namespace, its sub-processes can exit without any problem.


To back this very helpful answer up with some man page references: [`man 2 unshare`](http://man7.org/linux/man-pages/man2/unshare.2.html) says about `CLONE_NEWPID`: _Unshare the PID namespace, so that the calling process has a new PID namespace for its children which is not shared with any previously existing process. The calling process is not moved into the new namespace. **The first child created by the calling process will have the process ID 1** and will assume the role of init(1) in the new namespace._

Your `ps` probably works by reading `/proc`, and you still have your parent namespace's `/proc` mounted. Use `--mount-proc`.

