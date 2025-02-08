# 5.1 socket 地址 API
## 5.1.1 主机字节序和网络字节序
现代 PC 大多采用小端字节序，因此小端字节序被称为主机字节序。大端字节序称为网络字节序。需要指出的是，即使是同一台机器上的两个进程（比如一个由 C 语言编写，另一个由 JAVA 编写）通信，也要考虑字节序的问题。

Linux 提供了如下 4 个函数来完成主机字节序和网络字节序之间的转换：
```
#inclde <netinet/in.h>
unsigned long int htol(unsigned long int hostlong);
unsigned short int htons(unsigned short int hostshort);
unsigned long int ntohl(unsigned long int netlong);
unsigned short int ntohs(unsigned short int netshort);
```

htonl 表示 "host to network long"， 即将长整型（32bit）的主机字节序数据转化为网络字节序数据。

## 5.1.2 通用 socket 地址
socket 网络编程接口中表示 socket 地址的是结构体 sockaddr，其定义如下：
```
#include <bits/socket.h>
struct sockaddr{
	sa_family sa_family;
	char sa_data[14];
};
```

sa_family 成员是地址族类型（sa_family_t）的变量。地址族类型通常与协议族类型对应。常见的协议族（protocol family，也称 domain）。
![[Pasted image 20250208130843.png]]

sa_data 成员用于存放 socket 地址值。不同的协议族的地址值具有不同的含义和长度。
![[Pasted image 20250208131006.png]]
由表可见，14字节的sa_data 无法容纳多数协议族的地址值。因此，Linux定义了下面这个新的socket地址结构体。
```
#include <bits/socket.h>
struct sockaddr_storage{
	sa_family sa_family;
	unsigned long int __ss_align;
	char __ss_padding[128-sizeof(__ss_align)];
};
```
这个结构体不仅提供了足够大的空间用于存放地址值，而且是内存对齐的。

## 5.1.3 专用 socket 地址
Linux 为各个协议族提供了专门的 socket 地址结构体。

UNIX 本地域协议族使用如下专用 socket 地址结构体：
```
#include <sys/un.h>
struct sockaddr_un{
	sa_family sin_family;
	char sun_path[108];
};
```

TCP/IP 协议族有 sockaddr_in 和 sockaddr_in6 两个专用 socket 地址结构体，它们分别用于 IPv4 和 v6：
![[Pasted image 20250208131641.png]]
所有专用 socket 地址（以及 sockaddr_storage）类型的变量在实际使用时都需要转化为通用 socket 地址类型 sockaddr（强制转换），因为所有 socket 编程接口使用的地址参数的类型都是 sockaddr。

## 5.1.4 IP 地址转换函数
下面 3 个函数可用于用点分十进制字符串表示的 IPv4 地址和用网络字节序整数表示的 IPv4 地址之间的转换：
![[Pasted image 20250208132047.png]]
inet_addr 函数将点分十进制表示的 IPv4 地址转化为用网络字节序整数表示的 IPv4 地址。失败返回 INADDR_NONE。

inet_aton 将转化结果存储于参数 inp 指向的地址结构中。它成功时返回 1。

inet_ntoa 函数将用网络字节序整数表示的 IPv4 地址转化为用点分十进制字符串表示的 IPv4 地址。该函数内部用一个静态变量存储转化结果，函数的返回值指向该静态内存，因此 inet_ntoa 是不可重入的。

`inet_ntoa()` 是一个不可重入的函数。不可重入性的意思是，在多个线程或递归调用的环境下，如果在同一个时间调用了该函数，它可能会因为共享静态数据而产生不预期的行为。

具体地，`inet_ntoa()` 使用一个静态缓冲区来保存转换后的 IP 地址字符串。这意味着每次调用 `inet_ntoa()` 时，它会重写这个静态缓冲区的内容。因为这个静态缓冲区是全局的，多个并发调用可能会互相覆盖，导致返回的字符串被破坏。

下面这对更新的函数也能完成和前面 3 个函数同样的功能，并且它们同时适用于 IPv4 地址和 IPv6 地址：
![[Pasted image 20250208133524.png]]

inet_pton 函数将用字符串表示的 IP 地址 src 转换成用网络字节序整数表示的 IP 地址，并把结果存储于 dst 指向的内存中。其中，af 参数指定地址族，可以是 AF_INET 或者 AF_INET6。inet_pton 成功时返回1，失败则返回0，并设置 errno。

inet_ntop 函数进行相反的转换，前三个参数的含义与 inet_pton 的参数相同，最后一个参数 cnt 指定目标存储单元的大小。下面的两个宏能帮助我们指定这个大小（分别用于 IPv4 和 v6）：
![[Pasted image 20250208133901.png]]
inet_ntop 成功时返回目标存储单元的地址，失败则返回 NULL 并设置 errno。

# 5.2 创建 socket
socket 就是可读、可写、可控制、可关闭的文件描述符。下面的 socket 系统调用可创建一个 socket：
![[Pasted image 20250208134016.png]]
domain 参数告诉系统使用哪个底层协议族。type 参数指定服务类型。服务类型主要有 SOCK_STREAM 服务（流服务）和 SOCK_UGRAM（数据报）服务。流服务表示传输层使用 TCP 协议。

protocol 参数是在前两个参数构成的协议集合下，再选择一个具体的协议。几乎在所有情况下，设置为0。

socket 系统调用成功时返回一个 socket 文件描述符，失败则返回 -1 并设置 errno。

# 5.3 命名 socket
在网络编程中，**协议族**（Protocol Family）和**地址族**（Address Family）是两个密切相关但有所不同的概念。它们通常在创建和配置套接字时被使用。理解它们的区别有助于更好地理解套接字编程中的各个配置选项。

### 1. **协议族（Protocol Family）**

协议族定义了套接字所使用的协议类型，它决定了套接字能进行什么样的通信。协议族通常是一个更高层的抽象，用来指定套接字应该如何在网络上进行通信。

常见的协议族包括：

- **`AF_INET`**：IPv4 协议族，支持使用 IPv4 地址进行通信。
- **`AF_INET6`**：IPv6 协议族，支持使用 IPv6 地址进行通信。
- **`AF_UNIX`**（或 `AF_LOCAL`）：本地通信协议族，支持 Unix 域套接字，主要用于同一台机器上的进程间通信（IPC）。
- **`AF_NETLINK`**：用于与 Linux 内核进行通信的协议族。
- **`AF_PACKET`**：用于链路层通信，直接访问网络接口设备的协议族。

**协议族**主要决定了你使用的网络协议类型，它与后续的 `protocol` 参数相关。不同协议族支持不同类型的通信协议。例如，`AF_INET` 适用于 IPv4，而 `AF_INET6` 适用于 IPv6。

### 2. **地址族（Address Family）**

地址族是指套接字连接时使用的具体地址格式。它决定了套接字所用的地址类型，和通信的具体地址结构相关。地址族通常是与协议族紧密绑定的。

常见的地址族包括：

- **`AF_INET`**：IPv4 地址族，地址通常是 32 位的点分十进制表示（如 `192.168.1.1`）。
- **`AF_INET6`**：IPv6 地址族，地址通常是 128 位的十六进制表示（如 `2001:db8::1`）。
- **`AF_UNIX`**：Unix 域套接字地址族，地址通常是文件路径（如 `/tmp/socketfile`）。

### 区别总结：

- **协议族**：决定了你所使用的通信协议类型（如 TCP、UDP、IPv4、IPv6 等），是套接字通信的协议类型定义。
- **地址族**：决定了你使用的地址类型（如 IPv4 地址、IPv6 地址、Unix 域路径等），即套接字使用的具体地址格式。

### 示例：

在创建一个 IPv4 套接字时：

```c
int sockfd = socket(AF_INET, SOCK_STREAM, 0);
```

- `AF_INET`：协议族，表示使用 IPv4。
- `SOCK_STREAM`：套接字类型，表示使用流式套接字（如 TCP）。

而当你使用 `bind()` 函数绑定套接字时，会提供一个与协议族匹配的地址结构：

```c
struct sockaddr_in addr;
addr.sin_family = AF_INET;  // 地址族
addr.sin_port = htons(8080);
addr.sin_addr.s_addr = INADDR_ANY;
bind(sockfd, (struct sockaddr *)&addr, sizeof(addr));
```

- `sin_family` 也是地址族，表示这是一个 IPv4 地址。

### 结论：

- **协议族**和**地址族**虽然在很多情况下都可能是相同的（例如，`AF_INET` 对应 IPv4 地址和协议），但它们的概念是不同的。协议族关心的是通信协议，而地址族则指定了使用的具体地址格式。

创建 socket 时，指定了地址族，但是并未指定使用该地址族中的哪个具体 socket 地址。将一个 socket 与 socket 地址绑定称为给 socket 命名。在服务器程序中，通常要命名 socket。命名 socket 的系统调用是 bind。
![[Pasted image 20250208135056.png]]

bind 将 my_addr 所指的 socket 地址分配给未命名的 sockfd 文件描述符，addrlen 参数指出该 socket 地址的长度。

bind 成功时返回 0，失败则返回 -1 并设置 errno。其中1两种常见的 errno 是 EACCES 和 EADDRINUSE，它们的含义分别是：
- EACCES，被绑定的地址是受保护的地址，仅超级用户能够访问。比如普通用户将 socket 绑定到知名服务端口（端口号为 0~1023）上时，bind 将返回 EACCES 错误。
- EADDRINUSE，被绑定的地址正在使用中。比如将 socket 绑定到一个处于 TIME_WAIT 状态的 socket 地址。

# 5.4 监听 socket
socket 被命名之后，还需要创建一个监听队列以存放待处理的客户连接：
![[Pasted image 20250208135538.png]]
sockfd 参数指定被监听的 socket。backlog 参数提示内核监听队列的最大长度。

listen 成功返回 0，失败则返回 -1 并设置 errno。

# 5.5 接受连接
从 listen 监听队列中接受一个连接：
![[Pasted image 20250208140228.png]]

addr 参数用来获取被接受连接的远端 socket 地址。accept 成功时返回一个新的连接 socket，该 socket 唯一地标识了被接受地这个连接，服务器可通过读写该 socket 来与被接受连接对应的客户端通信。失败返回 -1 并设置 errno。

如果监听队列中处于 ESTABLISHED 状态的连接对应的客户端出现网络异常，那么服务器对这个连接执行的 accept 调用是否成功？

在 Kongming20 上运行该服务器程序，并在 ernest-laptop 上执行 telnet 命令来连接该服务器程序。具体操作过程如下：
![[Pasted image 20250208142159.png]]

![[Pasted image 20250208142256.png]]
![[Pasted image 20250208142303.png]]

# 5.6 发起连接
服务器通过 listen 调用来被动接受连接，那么客户端需要通过如下系统调用来主动与服务器建立连接：
![[Pasted image 20250208142818.png]]

# 5.7 关闭连接
关闭一个连接实际上就是关闭该连接对应的 socket。
![[Pasted image 20250208143018.png]]
fd 参数是待关闭的 socket。不过，close 系统调用并非总是立即关闭一个连接，而是将 fd 的引用计数减 1。只有当 fd 的引用计数为 0 时，才真正关闭连接。多进程中，一次 fork 默认将使父进程中打开的 socket 的引用计数加 1。

可以使用终止连接
![[Pasted image 20250208143226.png]]
![[Pasted image 20250208143312.png]]

# 5.8 数据读写
