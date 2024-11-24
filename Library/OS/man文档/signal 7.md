# NAME
signal - overview of signals

# DESCRIPTION
## Signal dispositions
Each signal has a current disposition, which determines how the process bahaves when it is delivered the signal.

The entries in the "Action" column of the table below specify the default disposition for each signal, as follows:

|      |                                                                           |
| ---- | ------------------------------------------------------------------------- |
| Ign  | Default action is to ignore the signal                                    |
| Core | Default action is to terminate the process and dump core (see [[core 5]]) |
| Term | Default action is to terminate the process                                |
| Stop | Default action is to stop the process                                     |
| Cont | Default action is to continue the process if it is currently stopped      |

A process can change the disposition of a signal using [[sigaction 2]] or [[signal 2]]. (The latter is less portable when establishing a signal handler) Using these system calls, a process can elect one of the following behaviors to occur on delivery of the signal: perform the default action; ignore the signal; or catch the signal with a signal handler, a programmer-defined function that is automatically invoked when the signal is delivered.

By default, a signal handler is invoked on the normal process stack. It is possible to arrange that the signal handler uses an alternate stack; see [[sigaltstack 2]] for  a discussion of how to do this and when it might be useful.

默认情况下，信号处理程序在正常的进程栈上被调用。不过，可以设置使信号处理程序使用备用栈；有关如何实现这一点以及在何种情况下可能有用，请参见 `sigaltstack(2)` 的相关讨论。

The signal disposition is a per-process attribute: in a multithreaded application, the disposition of a particular signal is the same for all threads.

信号处理程序是一个进程级别的属性：在一个多线程应用程序中，特定信号的处理程序对于所有线程是相同的。

A child created via [[fork 2]] inherits a copy of its parent's signal dispositions. During an [[execve 2]], the dispositions of handled signals are reset to the default; the dispositions of ignored signals are left unchanged.

## Sending a signal
The following system calls and library functions allow the caller to send a signal:

|                 |                                                                                                                              |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| raise(3)        | Sends a signal to the calling thread.                                                                                        |
| kill(2)         | Sends a signal to a specified process, to all members of a specified process group, or to all processes on the system        |
| killpg(3)       | Sends a signal to all of the members of a specified process group                                                            |
| pthread_kill(3) | Sends a signal to a specified POSIX thread in the same process as the caller                                                 |
| tgkill(2)       | Sends a signal to a specified thread within a specific process. (This is the system call used to implement pthread_kill(3).) |
| sigqueue(3)     | Sends a real-time signal with accompanying data to a specified process                                                       |

## Waiting for a signal to be caught
The following system calls suspend execution of the calling thread until a signal is caught (or an unhandled signal terminates the process):

|               |                                                                                                         |
| ------------- | ------------------------------------------------------------------------------------------------------- |
| pause(2)      | Suspends execution until any signal is caught                                                           |
| sigsuspend(2) | Temporarily changes the signal mask and suspends execution until one of  the unmasked signals is caught |

## Synchronously accepting a signal
不是同步接收信号，是异步接收信号
It is possible to synchronously accept the signal, that is, to block execution until the signal is delivered, at which point the kernel returns information about the signal to the caller. There are two ways to do this:
- sigwaitinfo(2), sigtimedwait(2), and sigwait(3) suspend execution until one of the signals in a specified set is delivered. Each of these calls returns information about the delivered signal.
- signalfd(2) returns a file descriptor that can be used to read information about signals that are delivered. Each [[read 2]] from this file descriptor blocks until one of the signals in the set specified in the signalfd(2) call is delivered to the caller. The buffer returned by read(2) contains a structure describing the signal.

## Signal mask and pending signal
A signal may be blocked, which means that it will not be delivered until it is later unblocked. Between the time when it is generated and when it is delivered a signal is said to be pending.

Each thread in a process has an independent signal mask, which indicates the set of signals that the thread is currently blocking. A thread can manipulate its signal mask using pthread_sigmask(3). In a traditional single-threaded application, sigprocmask(2) can be used to manipulate the signal mask.

