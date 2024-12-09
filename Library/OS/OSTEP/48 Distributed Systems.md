The major one we focus on is failure.
# The Crux:
How to build systems that work when components fall? How can we build a working system out of parts that don't work correctly all the time?

**System performance** is often critical; with a network connecting our distributed system together, system designers must often think carefully about how to accomplish their given tasks, trying to reduce the number of messages sent and further make communication as efficient (low latency, high bandwidth) as possible.

**Security** is also important.

In this introduction, we'll cover the most basic aspect that is new in a distributed system: **communication**. Namely, how should machines within a distributed system communicate with one another? We'll start with the most basic primitives available, messages, and build a few higher-level primitives on top of them. As we said above, failure will be a central focus: how should communication layers handle failures?

# 48.1 Communication Basics
通信不可靠，经常会有丢包。

然而，更根本的是由于网络交换机、路由器或端点内缺乏缓冲而导致的数据包丢失。具体来说，即使我们可以保证所有链路正常工作，并且系统中的所有组件（交换机、路由器、终端主机）都按预期启动和运行，仍然可能出现丢失，原因如下。假设一个数据包到达路由器；要处理该数据包，必须将其放在路由器内存中的某个位置。如果许多这样的数据包同时到达，路由器内的内存可能无法容纳所有数据包。路由器此时唯一的选择是丢弃一个或多个数据包。同样的行为也会发生在终端主机上；当您向一台机器发送大量消息时，该机器的资源很容易不堪重负，因此再次出现数据包丢失。

Thus, packet loss is fundamental in networking. The question thus becomes: how should we deal with it?

# 48.2 Unreliable Communication Layers
One simple way is this: we don't deal with it. 因为有些应用程序知道如何处理数据包丢失，所以有时让它们与基本的不可靠消息层进行通信很有用，这是人们经常听到的端到端论点的一个示例（请参阅本章末尾的旁注）。这种不可靠层的一个很好的例子是当今几乎所有现代系统上都可用的 UDP/IP 网络堆栈。要使用 UDP，进程使用套接字 API 来创建通信端点；其他机器（或同一台机器）上的进程将 UDP 数据报发送到原始进程（数据报是固定大小的消息，最大大小为某个值）。

图 48.1 和 48.2 显示了基于 UDP/IP 构建的简单客户端和服务器。客户端可以向服务器发送消息，然后服务器回复。有了这个少量的代码，您就拥有了开始构建分布式系统所需的一切！

UDP 是不可靠通信层的一个很好的例子。如果您使用它，您将遇到数据包丢失（丢弃）并因此无法到达目的地的情况；发送方永远不会被告知丢失。但是，这并不意味着 UDP 根本无法防范任何故障。例如，UDP 包含一个校验和来检测某些形式的数据包损坏。

但是，由于许多应用程序只是想将数据发送到目的地而不担心数据包丢失，因此我们需要更多。具体来说，我们需要在不可靠的网络上进行可靠的通信。

# 48.3 Reliable Communication Layers
为了构建可靠的通信层，我们需要一些新的机制和技术来处理数据包丢失。让我们考虑一个简单的示例，其中客户端通过不可靠的连接向服务器发送消息。我们必须回答的第一个问题是：发送者如何知道接收者确实收到了消息？