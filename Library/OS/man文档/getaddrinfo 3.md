`getaddrinfo` 是一个用于解析主机名和服务名的函数，通常用于网络编程中进行域名解析。它是一个更现代的、比旧有的 `gethostbyname` 和 `getservbyname` 更为灵活的 API。它支持 IPv4 和 IPv6，并能够提供更多的信息，如地址的具体类型、端口等。

### 函数原型：

```c
#include<sys/types.h>
#include<sys/socket.h>
#include<netdb.h>
int getaddrinfo(const char *node, const char *service, 
                const struct addrinfo *hints, 
                struct addrinfo **res);
```

### 参数说明：

- **node**: 一个主机名或者地址串(IPv4的点分十进制串或者IPv6的16进制串)
- **service**: 服务名（如 "http" 或 "80"）。可以为 `NULL`，这时解析器仅查询主机地址。服务名可以是十进制的端口号，也可以是已定义的服务名称，如ftp、http等
- **hints**: 提供了期望的地址类型等过滤条件。是一个指向 `struct addrinfo` 的指针，其中的各个字段可以设置期望的地址族（`AF_INET` 或 `AF_INET6`）、协议（`IPPROTO_TCP` 等），也可以指定是否需要 IPv6 地址。
- **res**: 输出参数，指向 `struct addrinfo` 指针，该结构体链表包含解析的结果。

### 返回值：

- 返回 0 表示成功。
- 非零值表示失败，可以通过 `gai_strerror()` 获取错误字符串。

### `struct addrinfo` 结构：

该结构用于存储解析结果，每个节点包含了一个网络地址及其附加信息。

```c
struct addrinfo {
    int ai_flags;          // 标志位（如 AI_PASSIVE, AI_CANONNAME 等）
    int ai_family;         // 地址族（如 AF_INET, AF_INET6 等）
    int ai_socktype;       // 套接字类型（如 SOCK_STREAM, SOCK_DGRAM 等）
    int ai_protocol;       // 协议（如 IPPROTO_TCP, IPPROTO_UDP 等）
    size_t ai_addrlen;     // 地址长度
    char *ai_canonname;    // 完整的主机名
    struct sockaddr *ai_addr;  // 地址信息
    struct addrinfo *ai_next;  // 下一个地址信息
};
```

### 示例代码：

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <netdb.h>
#include <unistd.h>

int main() {
    struct addrinfo hints, *res;
    int err;

    // 设置 hints 结构体
    memset(&hints, 0, sizeof(hints));
    hints.ai_family = AF_INET; // IPv4
    hints.ai_socktype = SOCK_STREAM; // TCP

    // 获取地址信息
    err = getaddrinfo("www.example.com", "http", &hints, &res);
    if (err != 0) {
        fprintf(stderr, "getaddrinfo: %s\n", gai_strerror(err));
        return 1;
    }

    // 输出地址信息
    char addrstr[100];
    void *addr;
    struct sockaddr_in *ipv4 = (struct sockaddr_in *)res->ai_addr;
    addr = &(ipv4->sin_addr);
    inet_ntop(res->ai_family, addr, addrstr, sizeof(addrstr));
    printf("Address: %s\n", addrstr);

    // 释放地址信息
    freeaddrinfo(res);
    return 0;
}
```

### 错误处理：

`getaddrinfo` 返回一个错误代码，如果失败，可以通过 `gai_strerror` 函数获取错误描述：

```c
char *gai_strerror(int errcode);
```

例如：

```c
int err = getaddrinfo("nonexistenthost", NULL, NULL, NULL);
if (err != 0) {
    printf("Error: %s\n", gai_strerror(err));
}
```

# 介绍
Given node and service, which identify an Internet host and a service, getaddrinfo() returns one or more addrinfo structures, each of which contains an Internet address that can be specified in a call to [[bind 2]] or [[connect 2]].

The hints argument points to an addrinfo structure that specifies criteria for selecting the socket address structures returned in the list pointed to by res. If hints is not NULL, it points to an addrinfo structure whose ai_family, ai_socktype, and ai_protocol specify criteria that limit the set of socket address returned by getaddrinfo(), as follows:
- ai_family: AF_INET, AF_INET6, the value AF_UNSPEC （包括IPv4 or IPv6）
- ai_socktype: This field specifies the preferred socket type, for example SOCK_STREAM or SOCK_DGRAM. Specifying 0 in this field indicates that socket addresses of any type can be returned by getaddrinfo().
- ai_protocol: This field specifies the protocol for the returned socket addresses. 0 代表任意
- ai_flags: This field specifies additional options.
All the other fields in the structure pointed to by hints must contain either 0 or a null pointer, as appropriate.

node specifies either a numerical network address (for IPv4, numbers-and-dots notation as supported by [[inet_aton 3]]); for IPv6, hexadecimal string format as supported by [[inet_pton 3]]; or a network hostname, whose network addresses are looked up and resolved. If hints.ai_flags contains the AI_NUMERICHOST flag, then node must be a numerical network address. The AI_NUMERICHOST flag suppresses any potentially lengthy network host address lookups.

If the AI_PASSIVE flag is specified in hints.ai_flags, and node is NULL, then the returned socket addresses will be suitable for [[bind 2]]ing a socket that will [[accept 2]] connections. The returned socket address will contain the "wildcard address" (INADDR_ANY for IPv4 addresses, IN6ADDR_ANY_INIT for IPv6 address). The wildcard address is used by applications (typically servers) that intend to accept connnections on any of the host's network addresses. If node is not NULL, then the AI_PASSIVE flag is ignored.


---
Q1-Why it needs to return a result that points to a **linked list** of `addrinfo` structures? I mean that given a host and service, these is only one unique socket address, how could it be more than one valid socket address so that a linked list is needed?

Q2-the last parameter is `struct addrinfo **result`, why it is a pointer to a pointer? Why it is not `struct addrinfo *result`, and then `getaddrinfo` creates sth internally and let `result`(`struct addrinfo *`) point to it? someone says it is due to `getaddrinfo` call `malloc` internally, but I can also code like this
```c
int main() 
{
   char *ptr_main;
   test(ptr_main);
   free(ptr_main);
}

void test(char * ptr)
{ 
    ptr = malloc(10); 
}
```
so the parameter to the function is `char *ptr`, not `char **ptr`.

getaddrinfo() returns a list of address because an hostname can have more than an address. Think for example to those high traffic sites that need to distribute visitors through different IPs.

Since `getaddrinfo()`

combines the functionality provided by the [[gethostbyname 3]] and [[getserbyname 3]] functions into a single interface, but unlike the latter functions, `getaddrinfo()` is reentrant and allows programs to elimminate IPv4-versus-IPv6 dependencies.

It might trigger a DNS session to resolve an host name. In case of those aforementioned high traffic sites the same hostname will correspond to a list of actual addresses.

>`struct addrinfo **result`, why is it a pointer to a pointer?

In C a pointer to something is passing as a parameter of a function when it has to modify it. So, for example, in case you need to modify an integer, you pass `int *`. This particular kind of 
_modification_ is very common in C when you want to return something through a parameter; in our previous example we can return an extra integer by accessing the pointer passed as a parameter.

But what if a function wants to allocate something? It will result, internally in a `type * var=malloc()`, meaning that a pointer to `type` would be returned. And in order to return it as a parameter we need to pass a `type **` parameter.

Given a `type`, if you want to return it as a parameter you have to define it as a pointer to `type`.

In conclusion, in our case the function `getaddrinfo` needs to modify a variable which type is `struct addrinfo *`, so `struct addrinfo **` is the parameter type.

Just to mention the meaning of this parameter:

> The geraddrinfo() function allocates and initializes a linked list of addrinfo structures, one for each network address that matches node and service, subject to any restrictions imposed by
> hints, and returns a pointer to the start of the list in res.

As you can see we actually have an allocation inside the function. So this memory will need to be eventually freed:
>The `freeaddrinfO()` function frees the memory that was allocated for the dynamically allocated linked list res.

# Why not just `type *` parameter?
Your code example results in undefined behavior, and when I run it it caused a program crash.

**Why?** Is I wrote above, in C parameters are passed **by value**. It means that in case of a `func(int c)` function, called in this way
```c
int b = 1234;

funct(b);
```
the parameter `c` internally used by the function will be a _copy_ of `b`, and any change on it won't survive outside the function.

The same happens in case of `func(char * ptr)` (please note the huge spacing, to underline how the type is `char *` and the variable is `ptr`): any change on `ptr` won't survive outside the function. **You'll be able to change the memory it points, and these changes will be available after the function returns, but the variable passed as a parameter will be the same it was before the call to `func()`.

And what was the value of `ptr_main`, in your example, before `test` was called? We don't know, as the variable is uninitialized. So, the behavior is undefined.

If you still have doubts, here it is a program that demonstrates that the newly allocated address obtained by value cannot be accessed from outside `the function`:

```c
#include <stdlib.h>
#include <stdio.h>

void test(char * ptr)
{ 
    ptr = malloc(10); 

    printf("test:\t%p\n", ptr);
}

int main() 
{
    char *ptr_main = (char *) 0x7777;

    printf("main-1:\t%p\n", ptr_main);
    test(ptr_main);
    printf("main-2:\t%p\n", ptr_main);
}
```
**Output:**

```c
main-1: 0000000000007777
test:   0000000000A96D60
main-2: 0000000000007777
```

Even after the function call the value of `ptr_main` is the same it after I initialized it (`0x7777`).