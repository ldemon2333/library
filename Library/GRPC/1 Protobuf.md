# 概述
**Protocol buffers** 是一种语言无关、平台无关的可扩展机制或者说是**数据交换格式**，用于**序列化结构化数据**。与 XML、JSON 相比，Protocol buffers 序列化后的码流更小、速度更快、操作更简单。

# Protocol Compiler
protoc 用于编译 protocolbuf (.proto文件) 和 protobuf 运行时。

# Go plugins
出了安装 protoc 之外还需要安装各个语言对应的**编译插件**，我用的 Go 语言，所以还需要安装一个 Go 语言的编译插件。

```
go get google.golang.org/protobuf/cmd/protoc-gen-go
```

# Demo
```
//声明proto的版本 只有 proto3 才支持 gRPC
syntax = "proto3";
// 将编译后文件输出在 github.com/lixd/grpc-go-example/helloworld/helloworld 目录
option go_package = "github.com/lixd/grpc-go-example/helloworld/helloworld";
// 指定当前proto文件属于helloworld包
package helloworld;

// 定义一个名叫 greeting 的服务
service Greeter {
  // 该服务包含一个 SayHello 方法 HelloRequest、HelloReply分别为该方法的输入与输出
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}
// 具体的参数定义
message HelloRequest {
  string name = 1;
}

message HelloReply {
  string message = 1;
}

```
## protoc 编译
```
# Syntax: protoc [OPTION] PROTO_FILES
$ protoc --proto_path=IMPORT_PATH  --go_out=OUT_DIR  --go_opt=paths=source_relative path/to/file.proto

```

# 编译过程
可以把 protoc 的编译过程分成简单的两个步骤：
- 1）解析 .proto 文件，编译成 protobuf 的原生数据结构保存在内存中；
- 2）把 protobuf 相关的数据结构传递给相应语言的**编译插件**，由插件负责将接收到的 protobuf 原生结构渲染输出为特定语言的模板。
![[Pasted image 20250527232746.png]]
使用 protoc 编译生成对应源文件，具体命令如下:
```
protoc --go_out=. --go_opt=paths=source_relative \
    --go-grpc_out=. --go-grpc_opt=paths=source_relative \
   ./hello_world.proto

```
会在当前目录生成`hello_world.pb.go`和`hello_world_grpc.pb.go`两个文件。

# 2 Hello gRPC
## 概述
gRPC 是一个高性能、通用的开源 RPC 框架，其由 Google 主要面向移动应用开发并基于 HTTP/2 协议标准而设计，基于 ProtoBuf(Protocol Buffers) 序列化协议开发，且支持众多开发语言。

与许多 RPC 系统类似，gRPC 也是基于以下理念：**定义一个`服务`，指定其能够被远程调用的方法（包含参数和返回类型）。在服务端实现这个接口，并运行一个 gRPC 服务器来处理客户端调用。在客户端拥有一个`存根`能够像服务端一样的方法**。

gRPC 默认使用 **protocol buffers**，这是 Google 开源的一套成熟的结构数据序列化机制（当然也可以使用其他数据格式如 JSON）。

![[Pasted image 20250527235657.png]]
使用 gRPC 的 3个 步骤:

- 1）需要使用 protobuf 定义接口，即编写 .proto 文件;
- 2）然后使用 protoc 工具配合编译插件编译生成特定语言或模块的执行代码，比如 Go、Java、C/C++、Python 等。
- 3）分别编写 server 端和 client 端代码，写入自己的业务逻辑。

