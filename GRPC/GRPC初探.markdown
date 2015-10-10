# GRPC初探

标签（空格分隔）： GRPC

---
[TOC]

gRPC是一个高性能、通用的开源RPC框架，其由Google主要面向移动应用开发并基于HTTP/2协议标准而设计，基于ProtoBuf(Protocol Buffers)序列化协议开发，且支持众多开发语言。

## 三大特征
- 强大的IDLe特性
gRPC使用ProtoBuf来定义服务，ProtoBuf是由Google开发的一种数据序列化协议（类似于XML、JSON、hessian）。ProtoBuf能够将数据进行序列化，并广泛应用在数据存储、通信协议等方面。不过，当前gRPC仅支持 Protobuf ，且不支持在浏览器中使用。由于gRPC的设计能够支持支持多种数据格式，所以读者能够很容易实现对其他数据格式（如XML、JSON等）的支持。

定义服务的示例代码如下：

```
message HelloRequest {
  string greeting = 1;
}
message HelloResponse {
  string reply = 1;
}
service HelloService {
  rpc SayHello(HelloRequest) returns (HelloResponse);
}
```

- 支持多种语言
gRPC支持多种语言，并能够基于语言自动生成客户端和服务端功能库。目前，在GitHub上已提供了C版本grpc、Java版本grpc-java 和 Go版本grpc-go，其它语言的版本正在积极开发中，其中 grpc支持C、C++、Node.js、Python、Ruby、Objective-C、PHP和C#等语言，grpc-java已经支持Android开发。

- 基于HTTP/2标准设计
由于gRPC基于HTTP/2标准设计，所以相对于其他RPC框架，gRPC带来了更多强大功能，如双向流、头部压缩、多复用请求等。这些功能给移动设备带来重大益处，如节省带宽、降低TCP链接次数、节省CPU使用和延长电池寿命等。同时，gRPC还能够提高了云端服务和Web应用的性能。gRPC既能够在客户端应用，也能够在服务器端应用，从而以透明的方式实现客户端和服务器端的通信和简化通信系统的构建。

  gRPC已经应用在Google的云服务和对外提供的API中，其主要应用场景如下：
  - 低延迟、高扩展性、分布式的系统
  - 同云服务器进行通信的移动应用客户端
  - 设计语言独立、高效、精确的新协议
  - 便于各方面扩展的分层设计，如认证、负载均衡、日志记录、监控等

近日，gRPC开发团队宣布gRPC基于三条款BSD许可协议（BSD 3-Clause License）开源，相关代码已托管在GitHub上。当前已有Google和移动支付公司Square以及其他组织或个人为该项目贡献代码。

## gRPC是什么
使用gRPC，一个客户端应用程序可以直接调用另一台机器上的服务端应用程序的方法，就好像调用本地方法一样。这使得更容易创建分布式应用程序和服务。正如很多RPC系统，gRPC是基于定义服务，指定可以远程调用方法的参数和返回类型的概念。在服务器侧，服务端实现该接口并且返回gRPC的客户端来处理客户端调用请求。在客户端侧，客户端打桩，提供跟服务端一样的方法。

![](http://www.grpc.io/img/grpc_concept_diagram_00.png)

gRPC客户端和服务器端可以在各种环境运行并互相通讯 - 从谷歌内部的服务器到你自己的电脑桌面 - 并且可以任何GRPC支持的语言实现。举个例子，你很容易使用Java创建一个gRPC的服务器与使用Go、Python或者Ruby实现的客户端。此外，最新的谷歌的API将有gRPC版本的接口，轻松添加谷歌功能到你的应用程序中。

### 使用ProtoBuf
默认情况下gRPC使用协议缓冲区，协议缓冲区是谷歌结构化数据序列化的一种成熟的开源机制（尽管它可以用于其它数据格式的序列化，如JSON）。在下面的例子中将可以看到，使用.proto文件来定义gRPC服务，包括方法参数和返回指定为协议缓冲区消息类型的类型。[更多有关协议缓冲区的文档](https://developers.google.com/protocol-buffers/docs/overview)。

#### 协议缓冲区版本(Protocol buffer versions)
虽然协议缓冲区在开源社区中已经使用了一段时间，我们使用的协议缓冲区的新版本名为proto3，有更简单的语法，一些有用的新功能，并支持更多的语言。这是目前可作为JAVA，C ++，Java_nano（Android的Java），Python和Ruby的一个alpha版本。

一般情况下，虽然可以使用proto2（当前默认的协议缓冲区版本），但是建议使用proto3与gRPC，因为它可以让你使用全套GRPC支持的语言，以及避免proto2客户端与proto3服务器通讯的兼容性问题，反之亦然。

## gRPC，你好！
接下来，通过简单的例子来理解gRPC的工作原理。通过Hello World来学习简单的GRPC客户端 - 服务器端的应用程序：
- 创建一个协议缓冲区schema，并定义一个简单的RPC服务，包含一个Hello World方法。
- 创建一个服务器，用你想要的语言实现这个接口。
- 用你想要的语言创建一个客户端，并访问你的服务器。




