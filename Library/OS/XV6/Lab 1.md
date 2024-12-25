# pingpong
编写一个程序，使用 UNIX 系统调用通过一对管道在两个进程之间“乒乓”一个字节，每个方向一个。父进程应向子进程发送一个字节；子进程应打印“\<pid>: received ping”，其中 \<pid> 是其进程 ID，将管道上的字节写入父进程，然后退出；父进程应从子进程读取该字节，打印“\<pid>: received pong”，然后退出。您的解决方案应位于文件 user/pingpong.c 中。

## sol
pipe 是半双工通信，数据只能单向流动.

开两个 pipe ，一个pipe ，父读子写，一个父写子读。

先是父写一个字节，子读，子进程打印信息，然后子写一个字节个给父，父接收，打印信息。

# primes
Write a concurrent version of prime sieve using pipes. This idea is due to Doug McIlroy, inventor of Unix pipes. The picture halfway down [this page](http://swtch.com/~rsc/thread/) and the surrounding text explain how to do it. Your solution should be in the file user/primes.c.

使用管道编写素数筛的并发版本。这个想法来自 Unix 管道的发明者 Doug McIlroy。本页中间的图片和周围的文字解释了如何做到这一点。您的解决方案应该在文件 user/primes.c 中。

您的目标是使用 pipe 和 fork 来设置管道。第一个进程将数字 2 到 35 输入管道。对于每个素数，您将安排创建一个进程，该进程通过管道从其左侧邻居读取，并通过另一个管道写入其右侧邻居。由于 xv6 的文件描述符和进程数量有限，因此第一个进程可以在 35 处停止。

hints：
- *Be careful to close file descriptors that a process doesn't need*, because otherwise your program will run xv6 out of resources before the first process reaches 35.
- Once the first process reaches 35, it should *wait* until the entire pipeline terminates, including all children, grandchildren, &c. Thus the main primes process should only exit after all the output has been printed, and after all the other primes processes have exited.
- Hint: read returns zero when the write-side of a pipe is closed.
- It's simplest to directly write 32-bit (4-byte) ints to the pipes, rather than using formatted ASCII I/O.
- You should create the processes in the pipeline only as they are needed.

## 介绍
这种风格的并发编程之所以有趣，不是因为效率，而是因为清晰度。也就是说，人们普遍错误地认为并发编程只是提高性能的一种手段，例如重叠磁盘 I/O 请求、通过预取预期查询的结果来减少延迟或利用多个处理器。这些优势很重要，但与本文讨论无关。毕竟，它们可以通过其他风格实现，例如异步事件驱动编程。相反，我们对并发编程感兴趣，因为它提供了一种自然的抽象，可以使一些程序变得更简单。

## **Communicating Sequential Processes**
到 1978 年，在多处理器编程的背景下，已经提出了许多用于通信和同步的方法。共享内存是最常见的通信机制，信号量、临界区和监视器属于同步机制。C. A. R. Hoare 使用一种语言原语解决了这两个问题：同步通信。在 Hoare 的 CSP 语言中，进程通过从命名的非缓冲通道发送或接收值来进行通信。由于通道是非缓冲的，因此发送操作会阻塞，直到值传输到接收器，从而提供了一种同步机制。

Hoare 的一个例子是重新格式化 80 列卡片，以便在 125 列打印机上打印。在他的解决方案中，一个进程一次读取一张卡片，将分解的内容逐个字符地发送给第二个进程。第二个进程将 125 个字符组合起来，将这些字符组发送到行式打印机。这听起来微不足道，但在没有缓冲 I/O 库的情况下，单进程解决方案中涉及的必要簿记工作非常繁重。事实上，缓冲 I/O 库实际上只是这两种导出单字符通信接口的进程的封装。

另一个例子，Hoare 将其归功于 Doug McIlroy，考虑生成所有小于一千的素数。埃拉托斯特尼筛法可以通过执行以下伪代码的流程流水线来模拟：
![[Pasted image 20241210112703.png]]
A generating process can feed the numbers 2, 3, 4, ..., 1000 into the left end of the pipeline: the first process in the line eliminates the multiples of 2, the second eliminates the multiples of 3, the third eliminates the multiples of 5, and so on:

![[Pasted image 20241210112739.png]]

注意最开始的父进程要等待所有子进程exit才能exit

解决思路：采用递归，每次先尝试从左pipe中读取一个数，如果读不到说明已经到达终点，exit，否则再创建一个右pipe并fork一个子进程，将筛选后的数feed进这个右pipe。

每一个 stage 以当前数集中最小的数字作为素数输出（每个 stage 中数集中最小的数一定是一个素数，因为它没有被任何比它小的数筛掉），并筛掉输入中该素数的所有倍数（必然不是素数），然后将剩下的数传递给下一 stage。最后会形成一条子进程链，而由于每一个进程都调用了 `wait(0);` 等待其子进程，所以会在最末端也就是最后一个 stage 完成的时候，沿着链条向上依次退出各个进程。

# find 
Write a simple version of the UNIX find program: find all the files in a directory tree with a specific name.

hints:
- Look at user/ls.c to see how to read directories.
- Use recursion to allow find to descend into sub-directories.
- Don't recurse into "." and "..".
- Changes to the file system persist across runs of qemu; to get a clean file system run `make clean` and then `make qemu`.
- You'll need to use C strings. Have a look at K&R (the C book), for example Section 5.5.
- Note that == does not compare strings like in Python. Use strcmp() instead.
- Add the program to UPROGS in Makefile.

# xargs 
编写 UNIX xargs 程序的简单版本：从标准输入读取行并对每一行运行一个命令，并将该行作为命令的参数。

请注意，此处的命令是“echo bye”，附加参数是“hello too”，因此命令为“echo bye hello too”，输出“bye hello too”。

请注意，UNIX 上的 xargs 进行了优化，每次将多个参数提供给命令。我们不希望您进行此优化。要使 UNIX 上的 xargs 按照我们希望的方式运行，请在运行它时将 -n 选项设置为 1。例如

![[Pasted image 20241210151528.png]]

hints：
- Use fork and exec to invoke the command on each line of input. Use wait in the parent to wait for the child to complete the command.
- To read individual lines of input, read a character at a time until a newline ('\n') appears.
- kernel/param.h declares MAXARG, which may be useful if you need to declare an argv array.
- Add the program to UPROGS in Makefile.
- Changes to the file system persist across runs of qemu; to get a clean file system run make clean and then make qemu.
