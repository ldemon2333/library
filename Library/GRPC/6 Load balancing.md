# Name resolving

This examples shows how `ClientConn` can pick different name resolvers.

## What is a name resolver

A name resolver can be seen as a `map[service-name][]backend-ip`. It takes a service name, and returns a list of IPs of the backends. A common used name resolver is DNS.

In this example, a resolver is created to resolve `resolver.example.grpc.io` to `localhost:50051`.

## Explanation
The echo server is serving on ":50051". Two clients are created, one is dialing to `passthrough:///localhost:50051`, while the other is dialing to `example:///resolver.example.grpc.io`. Both of them can connect the server.

Name resolver is picked based on the `scheme` in the target string. See [https://github.com/grpc/grpc/blob/master/doc/naming.md](https://github.com/grpc/grpc/blob/master/doc/naming.md) for the target syntax.

The first client picks the `passthrough` resolver, which takes the input, and use it as the backend addresses.

The second is connecting to service name `resolver.example.grpc.io`. Without a proper name resolver, this would fail. In the example it picks the `example` resolver that we installed. The `example` resolver can handle `resolver.example.grpc.io` correctly by returning the backend address. So even though the backend IP is not set when ClientConn is created, the connection will be created to the correct backend.

gRPC 中的默认 name-system 是 DNS，同时在客户端以插件形式提供了自定义 name-system 的机制。

gRPC NameResolver 会根据 name-system 选择对应的解析器，用以解析用户提供的服务器名，最后返回具体地址列表（IP+端口号）。

例如：默认使用 DNS name-system，我们只需要提供服务器的域名即端口号，NameResolver 就会使用 DNS 解析出域名对应的 IP 列表并返回。

![[Pasted image 20250606000420.png]]

- 1）客户端启动时，注册自定义的 resolver 。
    - 一般在 init() 方法，构造自定义的 resolveBuilder，并将其注册到 grpc 内部的 resolveBuilder 表中（其实是一个全局 map，key 为协议名，value 为构造的 resolveBuilder）。
- 2）客户端启动时通过自定义 Dail() 方法构造 grpc.ClientConn 单例
    - grpc.DialContext() 方法内部解析 URI，分析协议类型，并从 resolveBuilder 表中查找协议对应的 resolverBuilder。
    - 找到指定的 resolveBuilder 后，调用 resolveBuilder 的 Build() 方法，构建自定义 resolver，同时开启协程，通过此 resolver 更新被调服务实例列表。
    - Dial() 方法接收主调服务名和被调服务名，并根据自定义的协议名，基于这两个参数构造服务的 URI
    - Dial() 方法内部使用构造的 URI，调用 grpc.DialContext() 方法对指定服务进行拨号
- 3）grpc 底层 LB 库对每个实例均创建一个 subConnection，最终根据相应的 LB 策略，选择合适的 subConnection 处理某次 RPC 请求。

# Load balancing

This examples shows how `ClientConn` can pick different load balancing policies.

Note: to show the effect of load balancers, an example resolver is installed in this example to get the backend addresses. It's suggested to read the name resolver example before this example.

## Explanation

Two echo servers are serving on ":50051" and ":50052". They will include their serving address in the response. So the server on ":50051" will reply to the RPC with `this is examples/load_balancing (from :50051)`.

Two clients are created, to connect to both of these servers (they get both server addresses from the name resolver).

Each client picks a different load balancer (using `grpc.WithDefaultServiceConfig`): `pick_first` or `round_robin`. (These two policies are supported in gRPC by default. To add a custom balancing policy, implement the interfaces defined in [https://godoc.org/google.golang.org/grpc/balancer](https://godoc.org/google.golang.org/grpc/balancer)).

Note that balancers can also be switched using service config, which allows service owners (instead of client owners) to pick the balancer to use. Service config doc is available at [https://github.com/grpc/grpc/blob/master/doc/service_config.md](https://github.com/grpc/grpc/blob/master/doc/service_config.md).

### pick_first

The first client is configured to use `pick_first`. `pick_first` tries to connect to the first address, uses it for all RPCs if it connects, or try the next address if it fails (and keep doing that until one connection is successful). Because of this, all the RPCs will be sent to the same backend. The responses received all show the same backend address.

```
this is examples/load_balancing (from :50051)
this is examples/load_balancing (from :50051)
this is examples/load_balancing (from :50051)
this is examples/load_balancing (from :50051)
this is examples/load_balancing (from :50051)
this is examples/load_balancing (from :50051)
this is examples/load_balancing (from :50051)
this is examples/load_balancing (from :50051)
this is examples/load_balancing (from :50051)
this is examples/load_balancing (from :50051)
```

### round_robin

The second client is configured to use `round_robin`. `round_robin` connects to all the addresses it sees, and sends an RPC to each backend one at a time in order. E.g. the first RPC will be sent to backend-1, the second RPC will be sent to backend-2, and the third RPC will be sent to backend-1 again.

```
this is examples/load_balancing (from :50051)
this is examples/load_balancing (from :50051)
this is examples/load_balancing (from :50052)
this is examples/load_balancing (from :50051)
this is examples/load_balancing (from :50052)
this is examples/load_balancing (from :50051)
this is examples/load_balancing (from :50052)
this is examples/load_balancing (from :50051)
this is examples/load_balancing (from :50052)
this is examples/load_balancing (from :50051)
```

Note that it's possible to see two continues RPC sent to the same backend. That's because `round_robin` only picks the connections ready for RPCs. So if one of the two connections is not ready for some reason, all RPCs will be sent to the ready connection.


# 概述
gRPC 负载均衡包括客户端负载均衡和服务端负载均衡两种方向。本文主要介绍的是客户端负载均衡。

gRPC 的客户端负载均衡主要分为两个部分：

- 1）Name Resolver
- 2）Load Balancing Policy


# LB Policy
具体可以参考[官方文档-Load Balancing Policy](https://github.com/grpc/grpc/blob/master/doc/load-balancing.md)

常见的 gRPC 库都内置了几个负载均衡算法，比如 [gRPC-Go](https://github.com/grpc/grpc-go/tree/master/examples/features/load_balancing#pick_first) 中内置了`pick_first`和`round_robin`两种算法。

- pick_first：尝试连接到第一个地址，如果连接成功，则将其用于所有RPC，如果连接失败，则尝试下一个地址（并继续这样做，直到一个连接成功）。
- round_robin：连接到它看到的所有地址，并依次向每个后端发送一个RPC。例如，第一个RPC将发送到backend-1，第二个RPC将发送到backend-2，第三个RPC将再次发送到backend-1。

本例中我们会分别测试两种负载均衡策略的效果。


本文介绍的 负载均衡属于 **客户端负载均衡**，需要在客户端做较大改动，因为 gRPC-go 中已经实现了对应的代码，所以使用起来还是很简单的。

gRPC 内置负载均衡实现：

- 1）根据提供了服务名，使用对应 name resolver 解析获取到具体的 ip+端口号 列表
- 2）根据具体服务列表，分别建立连接
    - gRPC 内部也维护了一个连接池
- 3）根据负载均衡策略选取一个连接进行 rpc 请求

比如之前的例子，服务名为`example:///lb.example.grpc.lixueduan.com`，使用自定义的 name resolver 解析出来具体的服务列表为`localhost:50051,localhost:50052`.

然后调用 dial 建立连接时会分别与这两个服务建立连接。最后根据负载均衡策略选择一个连接来发起 rpc 请求。所以 pick_first会一直请求50051服务，而 round_robin 会交替请求 50051和50052。


# Custom Load Balancer
客户端配置为使用 custom_round_robin。custom_round_robin 会为每个接收到的端点创建一个 pick first 子节点。它会等待两个 pick first 子节点都准备就绪，然后遵从第一个 pick first 子节点的选取器，选择连接到 localhost:20000。except every chooseSecond times，它会遵从第二个 pick first 子节点的选取器，选择连接到 localhost:20001（反之亦然）。

custom_round_robin 被编写为一个委托策略，它包装了 pick_first 负载均衡器，每个接收到的端点都有一个。这是用户编写的自定义负载均衡器的预期指定方式，因为 pick first 将包含许多有用的功能，例如粘性瞬时故障、Happy Eyeballs 和健康检查。