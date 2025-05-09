到目前为止，我们在写并发时，好像构建并发应用程序的唯一方法是使用线程。就像生活中的许多事情一样，这并不完全正确。具体来说，在基于 GUI 的应用程序以及某些类型的互联网服务器中，经常使用不同的并发编程风格。这种风格称为基于事件的并发，在某些现代系统中很流行，包括服务器端框架，如 node.js，但它的根源在于我们将在下面讨论的 C/UNIX 系统。

基于事件的并发解决的问题有两个方面。首先，在多线程应用程序中正确管理并发可能具有挑战性；正如我们所讨论的，可能会出现丢失锁、死锁和其他令人讨厌的问题。第二，在多线程应用程序中，开发人员几乎无法控制在给定时刻安排的内容；相反，程序员只需创建线程然后希望底层操作系统以合理的方式在可用的 CPU 上调度它们。鉴于构建一个适用于所有工作负载的通用调度程序的难度，有时操作系统会以不太理想的方式调度工作。因此，我们有...

我们如何才能构建一个不使用线程的并发服务器，从而保持对并发的控制并避免一些似乎困扰多线程应用程序的问题？

--- 
要构建一个无线程的并发服务器，可以使用事件驱动模型和异步I/O。以下是一些实现方法：

1. **事件驱动模型**：
   - 使用单个主事件循环来管理多个连接。当一个事件发生时（例如接收到一个客户端请求），服务器可以根据事件类型调用相应的处理程序。
   - 利用事件通知机制（如 `select`、`poll`、`epoll`）来监视多个文件描述符的状态变化，从而实现非阻塞I/O操作。

2. **非阻塞I/O**：
   - 使用非阻塞的系统调用，如 `fcntl` 配置文件描述符为非阻塞模式。这允许服务器在发起I/O请求后继续处理其他事件，而不需要等待当前请求完成。

3. **异步I/O**：
   - 使用异步I/O接口，像是 `aio_read()` 和 `aio_write()`，允许程序发起I/O操作并立即返回，而不阻塞事件循环。

4. **状态机设计**：
   - 将每个连接的状态封装成一个状态机，这样可以在事件处理程序中维护状态，并根据当前状态决定接下来的操作。通过这种方式，可以有效管理连接的生命周期，而无需使用多线程。

5. **使用协程**：
   - 使用协程库（如 Python 的 `asyncio` 或 Go 的 goroutines），可以在单个线程中实现并发执行。协程通过 yield 机制暂停执行，允许其他协程运行，从而避免了多线程中的竞争和同步问题。

6. **消息队列**：
   - 引入消息队列来处理请求，将处理请求的工作分发到不同的事件循环或进程。通过这种方式，可以实现任务的异步处理，同时避免阻塞主事件循环。

---

# 33.1 The Basic Idea: An Event Loop
![[Pasted image 20241029182225.png]]
重要的是，当处理程序处理事件时，它是系统中发生的唯一活动；因此，决定接下来要处理哪个事件相当于调度。这种对调度的明确控制是基于事件的方法的基本优势之一。

但这个讨论给我们留下了一个更大的问题：基于事件的服务器究竟如何确定正在发生哪些事件，特别是关于网络和磁盘 I/O 的事件？具体来说，事件服务器如何判断消息是否已到达？

# 33.2 An Important API: select() (or poll())
考虑到基本事件循环，我们接下来必须解决如何接收事件的问题。在大多数系统中，可以通过 select() 或 poll() 系统调用获得基本 API。

这些接口使程序能够执行的操作很简单：检查是否有任何需要处理的传入 I/O。例如，假设网络应用程序（例如 Web 服务器）希望检查是否有任何网络数据包到达，以便为它们提供服务。

这些系统调用让您可以做到这一点。以 select() 为例。手册页（在 Mac 上）以这种方式描述 API：
![[Pasted image 20241029182658.png]]
手册页中的实际描述：select() 检查在 readfds、writefds 和 errorfds 中传递地址的 I/O 描述符集，以分别查看其中某些描述符是否已准备好读取、是否已准备好写入或是否有异常情况待处理。检查每个集合中的前 nfds 描述符，即检查描述符集合中从 0 到 nfds-1 的描述符。返回时，select() 将给定的描述符集替换为由已准备好执行请求操作的描述符组成的子集。select() 返回所有集合中就绪描述符的总数。

## 旁注：阻塞与非阻塞接口
阻塞（或同步）接口在返回给调用者之前完成所有工作；非阻塞（或异步）接口开始一些工作但立即返回，从而让需要完成的任何工作在后台完成。
阻塞调用的常见罪魁祸首是某种 I/O。例如，如果一个调用必须从磁盘读取才能完成，它可能会阻塞，等待已发送到磁盘的 I/O 请求返回。
非阻塞接口可用于任何编程风格（例如，使用线程），但在基于事件的方法中必不可少，因为阻塞的调用将停止所有进程。

关于 select() 的几点说明。首先，请注意，它允许您检查描述符是否可以读取和写入；前者让服务器确定新数据包已到达并需要处理，而后者让服务知道何时可以回复（即，出站队列未满）。

其次，请注意超时参数。这里的一个常见用法是将超时设置为 NULL，这会导致 select() 无限期阻塞，直到某个描述符准备就绪。但是，更强大的服务器通常会指定某种超时；一种常见的技术是将超时设置为零，从而使用对 select() 的调用立即返回。

poll() 系统调用非常相似。有关详细信息，请参阅其手册页或 Stevens 和 Rago。

无论如何，这些基本原语为我们提供了一种构建非阻塞事件循环的方法，该循环仅检查传入的数据包，从带有消息的套接字读取消息，并根据需要进行回复。

# 33.3 Using select()
![[Pasted image 20241029183323.png]]
This code is actually fairly simple to understand. After some initialization, the server enters an infinite loop. Inside the loop, it uses the FD ZERO() macro to first clear the set of file descriptors, and then uses FD SET() to include all of the file descriptors from minFD to maxFD in the set. This set of descriptors might represent, for example, all of the network sockets to which the server is paying attention. Finally, the server calls select() to see which of the connections have data available upon them. By then using FD ISSET() in a loop, the event server can see which of the descriptors have data ready and process the incoming data.

当然，真正的服务器会比这更复杂，并且需要在发送消息、发出磁盘 I/O 和许多其他细节时使用逻辑。

				提示：不要在基于事件的服务器中阻塞
基于事件的服务器可以对任务调度进行细粒度控制。但是，为了保持这种控制，永远不能进行任何阻止调用者执行的调用；不遵守此设计提示将导致基于事件的服务器被阻塞，客户端感到沮丧，并且会有人质疑您是否读过本书的这一部分。

# 33.4 Why Simpler? No Locks Needed
有了单个 CPU 和基于事件的应用程序，并发程序中发现的问题就不再存在。具体来说，由于一次只处理一个事件，因此无需获取或释放锁；基于事件的服务器不能被另一个线程中断，因为它绝对是单线程的。因此，线程程序中常见的并发错误不会在基于事件的基本方法中出现。

# 33.5 A Problem: Blocking System Calls
到目前为止，基于事件的编程听起来很棒，对吧？您编写一个简单的循环，并在事件发生时处理它们。您甚至不需要考虑锁定！但有一个问题：如果事件要求您发出可能阻塞的系统调用怎么办？

例如，假设一个请求从客户端进入服务器，从磁盘读取文件并将其内容返回给请求客户端（很像一个简单的 HTTP 请求）。为了处理这样的请求，某个事件处理程序最终必须发出 open() 系统调用来打开文件，然后是一系列 read() 调用来读取文件。当文件被读入内存时，服务器可能会开始将结果发送到客户端。

open() 和 read() 调用都可能向存储系统发出 I/O 请求（当所需的元数据或数据不在内存中时），因此可能需要很长时间才能提供服务。对于基于线程的服务器，这不是问题：当发出 I/O 请求的线程暂停（等待 I/O 完成）时，其他线程可以运行，从而使服务器能够取得进展。事实上，I/O 和其他计算的这种自然重叠使得基于线程的编程非常自然和简单。但是，对于基于事件的方法，没有其他线程可以运行：只有主事件循环。这意味着如果事件处理程序发出阻塞的调用，整个服务器都会这样做：阻塞直到调用完成。当事件循环阻塞时，系统处于空闲状态，因此可能会浪费大量资源。因此，我们有一条在基于事件的系统中必须遵守的规则：不允许阻塞调用。在基于事件驱动的系统，使用异步IO。

# 33.6 A Solution: Asynchronous I/O
为了克服这一限制，许多现代操作系统引入了向磁盘系统发出 I/O 请求的新方法，通常称为异步 I/O。这些接口使应用程序能够发出 I/O 请求并在 I/O 完成之前立即将控制权返回给调用者；附加接口使应用程序能够确定各种 I/O 是否已完成。

例如，让我们检查一下 Mac 上提供的接口（其他系统也有类似的 API）。这些 API 围绕一个基本结构展开，即 struct aiocb 或常用术语中的 AIO 控制块。该结构的简化版本如下所示（有关更多信息，请参阅手册页）：

![[Pasted image 20241029185627.png]]
要对文件发出异步读取，应用程序首先应使用相关信息填充此结构：要读取的文件的文件描述符（aio fildes）、文件内的偏移量（aio offset）以及请求的长度（aio nbytes），最后是读取结果应复制到的目标内存位置（aio buf）。

填充此结构后，应用程序必须发出异步调用来读取文件；在 Mac 上，此 API 只是异步读取 API：

![[Pasted image 20241029185712.png]]

此调用尝试发出 I/O；如果成功，它会立即返回，应用程序（即基于事件的服务器）可以继续其工作。
然而，我们必须解决最后一个难题。我们如何知道 I/O 何时完成，从而知道缓冲区（由 aio_buf 指向）现在包含请求的数据？
需要最后一个 API。在 Mac 上，它被称为 aio_error()（有点令人困惑）。API 如下所示：
![[Pasted image 20241029185829.png]]
此系统调用检查 aiocbp 引用的请求是否已完成。如果已完成，则例程返回成功（用零表示）；如果没有，则返回 EINPROGRESS。因此，对于每个未完成的异步 I/O，应用程序可以通过调用 aio_error() 定期轮询系统，以确定所述 I/O 是否已完成。

您可能已经注意到的一件事是，检查 I/O 是否已完成非常麻烦；如果程序在给定时间点发出了数十或数百个 I/O，它应该只是重复检查每个 I/O，还是先等待一段时间，还是……？

为了解决这个问题，一些系统提供了一种基于中断的方法。此方法使用 UNIX 信号在异步 I/O 完成时通知应用程序，从而无需反复询问系统。这种轮询与中断问题也出现在设备中，正如您将在 I/O 设备章节中看到（或已经看到）的那样。

在没有异步 I/O 的系统中，无法实现纯基于事件的方法。然而，聪明的研究人员已经得出了相当有效的方法。例如，Pai 等人 [PDZ99] 描述了一种混合方法，其中使用事件来处理网络数据包，并使用线程池来管理未完成的 I/O。阅读他们的论文了解详情。

# 33.7 Another Problem: State Management
基于事件的方法的另一个问题是，这种代码通常比传统的基于线程的代码更难编写。线程是有时间片轮转算法的。原因如下：当事件处理程序发出异步 I/O 时，它必须打包一些程序状态，以供下一个事件处理程序在 I/O 最终完成时使用；基于线程的程序不需要这项额外的工作，因为程序所需的状态位于线程的堆栈上。Adya 等人将这项工作称为手动堆栈管理，它是基于事件的编程的基础。

为了更具体地说明这一点，让我们看一个简单的例子，其中基于线程的服务器需要从文件描述符 (fd) 读取，并在完成后将从文件读取的数据写入网络套接字描述符 (sd)。代码（忽略错误检查）如下所示：
![[Pasted image 20241029190505.png]]
如您所见，在多线程程序中，执行此类工作很简单；当 read() 最终返回时，代码会立即知道要写入哪个套接字，因为该信息位于线程的堆栈上（在变量 sd 中）。

在基于事件的系统中，生活并不那么容易。要执行相同的任务，我们首先使用上面描述的 AIO 调用异步发出读取。假设我们随后使用 aio_error() 调用定期检查读取是否完成；当该调用通知我们读取已完成时，基于事件的服务器如何知道该做什么？

正如 Adya 等人所述，解决方案是使用一种称为延续的旧编程语言构造。虽然听起来很复杂，但这个想法相当简单：基本上，在某个数据结构中记录完成处理此事件所需的信息；当事件发生时（即磁盘 I/O 完成时），查找所需的信息并处理事件。

在这种特定情况下，解决方案是将套接字描述符 (sd) 记录在某种数据结构（例如哈希表）中，由文件描述符 (fd) 索引。当磁盘 I/O 完成时，事件处理程序将使用文件描述符查找延续，它将套接字描述符的值返回给调用者。此时（最后），服务器可以完成最后的工作，将数据写入套接字。

##  旁注：UNIX 信号
所有现代 UNIX 变体中都存在一个庞大而迷人的基础结构，即信号。简单地说，信号提供了一种与进程通信的方式。具体来说，可以将信号传递给应用程序；这样做会停止应用程序正在执行的任何操作，以运行信号处理程序，即应用程序中处理该信号的某些代码。完成后，进程将恢复其先前的行为。

每个信号都有一个名称，例如 HUP（挂断）、INT（中断）、SEGV（分段违规）等；有关详细信息，请参阅手册页。有趣的是，有时内核本身会发出信号。例如，当您的程序遇到分段违规时，操作系统会向其发送 SIGSEGV（在信号名称前面添加 SIG 很常见）；如果您的程序配置为捕获该信号，您实际上可以运行一些代码来响应此错误的程序行为（这有助于调试）。当信号发送到未配置为处理信号的进程时，将执行默认行为；对于 SEGV，进程将被终止。

这是一个进入无限循环的简单程序，但首先设置了一个信号处理程序来捕获 SIGHUP：

![[Pasted image 20241029190925.png]]
您可以使用 kill 命令行工具向其发送信号（是的，这是一个奇怪而激进的名字）。这样做会中断程序中的主 while 循环并运行处理程序代码 handle()：

![[Pasted image 20241029191005.png]]



# 33.8 What Is Still Difficult With Events
我们应该提到基于事件的方法还存在一些其他困难。例如，当系统从单个 CPU 转移到多个 CPU 时，基于事件的方法的一些简单性就消失了。具体来说，为了利用多个 CPU，事件服务器必须并行运行多个事件处理程序；这样做时，会出现常见的同步问题（例如临界区），并且必须采用常见的解决方案（例如锁）。因此，在现代多核系统上，不再可能进行没有锁的简单事件处理。

基于事件的方法的另一个问题是它不能很好地与某些类型的系统活动（例如分页）集成。例如，如果事件处理程序页面错误，它将被阻塞，因此服务器在页面错误完成之前不会取得进展。即使服务器的结构可以避免显式阻塞，但这种由于页面错误而导致的隐式阻塞很难避免，因此如果普遍存在，可能会导致严重的性能问题。

第三个问题是，随着时间的推移，基于事件的代码可能难以管理，因为各种例程的确切语义会发生变化。例如，如果例程从非阻塞变为阻塞，则调用该例程的事件处理程序也必须更改以适应其新性质，方法是将自身拆分为两部分。由于阻塞对于基于事件的服务器来说非常糟糕，因此程序员必须始终留意每个事件使用的 API 语义中的此类变化。

最后，尽管现在大多数平台上都可以实现异步磁盘 I/O，但要实现这一点却花了很长时间，而且它从未像您想象的那样以简单统一的方式与异步网络 I/O 完全集成。例如，虽然人们只想使用 select() 接口来管理所有未完成的 I/O，但通常需要将用于网络的 select() 与用于磁盘 I/O 的 AIO 调用进行某种组合。

# 33.9 总结
我们已经对基于事件的不同并发风格进行了简要介绍。基于事件的服务器将调度控制权交给应用程序本身，但这样做会付出一定代价，增加复杂性，并难以与现代系统的其他方面（例如分页）集成。由于这些挑战，没有一种方法是最好的；因此，在未来许多年里，线程和事件都可能作为两种不同的方法继续存在，以解决同一个并发问题。
