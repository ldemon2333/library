https://www.lixueduan.com/posts/grpc/04-encryption-tls/#2-server-side-tls

本文记录了gRPC 中如何通过 TLS 证书建立安全连接，让数据能够加密处理，包括证书制作和CA签名校验等

# 概述
gRPC 内置了以下 encryption 机制：
- 1）SSL / TLS：通过证书进行数据加密；
- 2）ALTS：Google开发的一种双向身份验证和传输加密系统。
    - 只有运行在 Google Cloud Platform 才可用，一般不用考虑。

**gRPC 中的连接类型一共有以下3种**：

- 1）insecure connection：不使用TLS加密
- 2）server-side TLS：仅服务端TLS加密
- 3）mutual TLS：客户端、服务端都使用TLS加密

本章将记录如何使用 `server-side TLS` 和`mutual TLS`来建立安全连接。

# server-side TLS
## 流程
服务端 TLS 具体包含以下几个步骤：
- 1）制作证书，包含服务端证书和 CA 证书；
- 2）服务端启动时加载证书；
- 3）客户端连接时使用CA 证书校验服务端证书有效性。


