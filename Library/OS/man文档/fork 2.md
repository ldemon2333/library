# 语法
```
#include <sys/types.h>
#include <unistd.h>

pid_t fork(void);
```
# 介绍
The child process and the parent process run in separate memory spaces. At the time of `fork()` both memory spaces have the same content. 

The child process is an exact duplicate of the parent process except for the following points:
- The child has its own unique process ID, and this PID does not match the ID of any existing process group [[setpgid 2]] or session.
- The child does not inherit its parent's memory locks ([[mlock 2 ]], [[mlockall 2]]).
- Process resource utilizations ([[getrusage 2]]) and CPU time counters ([[times 2]]) are reset to zero in the child.
- The child's set of pending signals is initially empty ([[sigpending 2]]).
- The child does not inherit semaphore adjustments from its parent ([[semop 2]]).
- The child does not inherit process-associated record locks from its parent ([[fcntl 2]]). 
- The child does not inherit timers from its parent.
- The child does not inherit outstanding asynchronous I/O operations from its parent, nor does it inherit any asynchronous I/O contexts from its parent.

Note the following further points:
- The child inherits copies of the parent's set of open file descriptors. Each file descriptor in the child refers to the same open file description ([[open 2]]) as the corresponding file descriptor in the parent. This means that the two file descriptors share open file status flags, file offset, and signal-driven I/O arrtibutes.
