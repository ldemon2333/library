gRPC 提供了 Interceptor 功能，包括客户端拦截器和服务端拦截器。可以在接收到请求或者发起请求之前优先对请求中的数据做一些处理后再转交给指定的服务处理并响应，很适合在这里处理验证、日志等流程。

gRPC-go 在 v1.28.0版本增加了多 interceptor 支持，可以在不借助第三方库（go-grpc-middleware）的情况下添加多个 interceptor 了。

在 gRPC 中，根据拦截的方法类型不同可以分为拦截 Unary 方法的**一元拦截器**，和作用于 Stream 方法的**流拦截器**。

同时还分为**服务端拦截器**和**客户端拦截器**，所以一共有以下4种类型:

- grpc.UnaryServerInterceptor
- grpc.StreamServerInterceptor
- grpc.UnaryClientInterceptor
- grpc.StreamClientInterceptor

# 定义
## 客户端拦截器
使用客户端拦截器 只需要在 Dial的时候指定相应的 DialOption 即可。

### Unary Interceptor


# 3 UnaryInterceptor
一元拦截器可以分为3个阶段：
- 1）预处理(pre-processing)
- 2）调用RPC方法(invoking RPC method)
- 3）后处理(post-processing)


