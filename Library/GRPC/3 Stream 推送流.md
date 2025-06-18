https://www.lixueduan.com/posts/grpc/03-stream/#6-bidirectionalstream

本文主要讲述了 gRPC 中的四种类型的方法使用，包括普通的 Unary API 和三种 Stream API：`ServerStreaming`、`ClientStreaming`、`BidirectionalStreaming`。

# 概述
gRPC 中的 Service API 有如下4种类型：

- 1）UnaryAPI：普通一元方法
- 2）ServerStreaming：服务端推送流
- 3）ClientStreaming：客户端推送流
- 4）BidirectionalStreaming：双向推送流

Unary API 就是普通的 RPC 调用，例如之前的 HelloWorld 就是一个 Unary API，本文主要讲解 Stream API。

Stream 顾名思义就是一种流，可以源源不断的推送数据，很适合大数据传输，或者服务端和客户端长时间数据交互的场景。Stream API 和 Unary API 相比，因为省掉了中间每次建立连接的花费，所以效率上会提升一些。


# ServerStream
服务端流：服务端可以发送多个数据给客户端。

## Server
注意点：可以多次调用 stream.Send() 来返回多个数据。


# 7. 小结

客户端或者服务端都有对应的 `推送`或者 `接收`对象，我们只要 不断循环 Recv(),或者 Send() 就能接收或者推送了！

> gRPC Stream 和 goroutine 配合简直完美。通过 Stream 我们可以更加灵活的实现自己的业务。如 订阅，大数据传输等。

**Client发送完成后需要手动调用Close()或者CloseSend()方法关闭stream，Server端则`return nil`就会自动 Close**。

**1）ServerStream**

- 服务端处理完成后`return nil`代表响应完成
- 客户端通过 `err == io.EOF`判断服务端是否响应完成

**2）ClientStream**

- 客户端发送完毕通过`CloseAndRecv关闭stream 并接收服务端响应
- 服务端通过 `err == io.EOF`判断客户端是否发送完毕，完毕后使用`SendAndClose`关闭 stream并返回响应。

**3）BidirectionalStream**

- 客户端服务端都通过stream向对方推送数据
- 客户端推送完成后通过`CloseSend关闭流，通过`err == io.EOF`判断服务端是否响应完成
- 服务端通过`err == io.EOF`判断客户端是否响应完成,通过`return nil`表示已经完成响应

通过`err == io.EOF`来判定是否把对方推送的数据全部获取到了。

客户端通过`CloseAndRecv`或者`CloseSend`关闭 Stream，服务端则通过`SendAndClose`或者直接 `return nil`来返回响应。




流控制是 gRPC 中的一项功能，可防止发送方在流上写入超出接收方处理能力的数据。此功能对客户端和服务器的行为相同。由于 gRPC-Go 使用阻塞式 API 进行流操作，因此当达到流的流控制限制时，只需阻止流上的发送操作即可实现流控制回推。当接收方从流中读取了足够的数据后，发送操作将自动解除阻塞。流控制根据连接的带宽延迟积 (BDP) 自动配置，以确保缓冲区具有在接收方以最大速度读取时允许流上最大吞吐量所需的最小大小。