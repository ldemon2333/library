IP 协议：
- IP 头部信息。IP 头部信息出现在每个 IP 数据报中，用于指定 IP 通信的源端 IP 地址、目的端 IP，指定 IP 分片和重组，以及指定部分通信行为。
- IP 数据报的路由和转发。

# 2.1 IP 服务的特点
IP 协议是 TCP/IP 协议族的动力，它为上层协议提供无状态、无连接、不可靠的服务。

无状态（stateless）是指 IP 通信双方不同步传输数据的状态信息，因此所有 IP 数据报的发送、传输和接收都是相互独立、没有上下文关系的。最大的缺点是无法处理乱序和重复的 IP 数据报。

不可靠指发送端的 IP 模块一旦检测到 IP 数据报发送失败，就通知上层协议发送失败，而不会试图重传。因此，使用 IP 服务的上层协议（比如 TCP 协议）需要自己实现数据确认、超时重传等机制以达到可靠传输的目的。

# 2.2 IPv4 头部结构
## 2.2.1 IPv4 头部结构
![[Pasted image 20250111215822.png]]

## 2.2.2 使用 tcpdump 观察 IPv4 头部结构
使用 tcpdump 抓取这个过程中 telnet 客户端程序和 telnet 服务器程序之间交换的数据包。
![[Pasted image 20250111220054.png]]
telnet 服务器程序使用的端口号是 23，而 telnet 客户端程序使用临时端口号 41621 与服务器通信。`length` 指出该 IP 数据报所携带的应用程序数据的长度。

这次抓包我们开启了 tcpdump 的 -x 选项，使之输出数据包的二进制码。此数据包共包含 60 字节，其中前 20 字节是 IP 头部，后 40 字节是 TCP 头部，不包含应用程序数据（length 值为0）。
![[Pasted image 20250111220853.png]]

# 2.3 IP 分片

# 2.4 IP 路由
## 2.4.1 IP 模块工作流程
![[Pasted image 20250111221106.png]]
从右往左分析。当 IP 模块接收到来自数据链路层的 IP 数据报时，它首先对该数据报的头部做 CRC 校验，确认无误之后就分析其头部的具体信息。

如果该 IP 数据报的头部设置了源站选路选项（松散源路由选择或严格源路由选择），则 IP 模块调用数据报转发子模块来处理该数据报。如果头部目标 IP 地址是本机的某个 IP 地址，或者是广播地址，即该数据报是发送给本机的，则 IP 模块就根据数据报头部中的协议字段来决定将它派发给哪个上层应用（分用）。如果 IP 模块发现这个数据报不是发送给本机的，则也调用数据报转发子模块来处理该数据报。

数据报转发子模块将首先检测系统是否允许转发，如果不允许，IP 模块就将数据报丢弃。如果允许，数据报转发子模块将对该数据报执行一些操作，然后将它交给 IP 数据报输出子模块。

IP 输出队列中存放的是所有等待发送的 IP 数据报。

虚线箭头显示了路由表更新的过程。这一过程是指通过路由协议或者 route 命令调整路由表，使之更适应最新的网络拓扑结构，称为 IP 路由策略。

## 2.4.2 路由机制
route 命令查看路由表
![[Pasted image 20250111222244.png]]
路由表中，第一项的目标地址是 default，即所谓的默认路由项。该项包含一个“G”标志，说明路由的下一跳目标是网关，其地址是 192.168.1.1（这是测试网络中路由器的本地 IP 地址）。另外一个路由项的目标地址是 192.168.1.0，它指的是本地局域网。该路由项的网关地址为 * ，说明数据报不需要路由中转，可以直接发送到目标机器。

IP 的路由机制：
1. 查找路由表中和数据报的目标 IP 地址完全匹配的主机 IP 地址。如果找到，就使用该路由项，没有转2
2. 查找路由表中和数据报的目标 IP 地址具有相同网络 ID 的网络 IP 地址。没找到转3.
3. 选择默认路由项，这通常意味着数据报的下一跳路由是网关。

因此，所有发送到 IP 地址为 192.168.1.* 的机器的 IP 数据报都可以直接发送到目标机器（匹配路由表第二项），而所有访问因特网的请求都将通过网关来转发。

## 2.4.3 路由表更新
route 命令可以修改路由表。
![[Pasted image 20250111223343.png]]

第 1 行表示添加主机 192.168.1.109（机器 Kongming20）对于的路由项。这样设置之后，所有从 ernest-laptop 发送到 Kongming20 的 IP 数据报将通过网卡 eth0 直接发送至目标机器的接收网卡。第 2 行表示删除网络 192.168.1.0 对应的路由项。这样测试机器 ernest-laptop 将无法访问该局域网上的任何其他机器（能访问到 Kongming20 是由于执行了上一条命令）。第 3 行表示删除默认路由项，这样做的后果是无法访问因特网。第 4 行重新设置默认路由项，这次其网关机器 Kongming20。
![[Pasted image 20250111224655.png]]

# 2.5 IP 转发
数据报转发子模块将对期望转发的数据报执行如下操作：
![[Pasted image 20250111225024.png]]

# 2.6 重定向
图 2-3 显示了 ICMP 重定向报文也能用于跟新路由表。

## 2.6.1 ICMP 重定向报文
![[Pasted image 20250111225217.png]]
ICMP 重定向报文的类型值是 5，代码字段有 4 个可选值，用来区分不同的重定向类型。主机重定向，其代码值为 1。

它给接收方提供：
- 引起重定向的 IP 数据报的源端 IP 地址
- 应该使用的路由器的 IP 地址
接收主机根据这两个信息就可以断点引起重定向的 IP 数据报应该使用哪个路由器来转发，并且以此来更新路由表。
![[Pasted image 20250111225719.png]]

# 2.7 IPv6 头部结构
## 2.7.1 IPv6 固定头部结构
![[Pasted image 20250111225836.png]]
![[Pasted image 20250111225858.png]]
