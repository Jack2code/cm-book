# OpenConfig分析

标签（空格分隔）： OpenConfig 

---
[TOC]

## OpenConfig
OpenConfig不是一个标准的组织，甚至连正式的组织都算不上，事实上，OpenConfig工作组当前是邀请加入的组织。

Google牵头和AT&T、微软、BT成立OpenConfig工作组，建立一套厂商中立的网络配置和策略模型，实现应用直接对网络进行编程而不需要人工干预。这个OpenConfig工作组在2014/10/26向IETF提交了BGP Configuration Model for Service Provider Networks初稿。这份文稿基于运营商和内容提供商运营需求，定义了一套配置和管理BGP协议、策略和运营方面的YANG数据模型。

Google希望开放其网络配置和拓扑模型，最终实现整个网络的抽象视图，实现应用自动触发的网络编程和配置的自动化，和厂商设备无关。

分析点评：

>SDN从DC聚焦控转分离、虚拟化对应到运营商公网一直找不到合适的突破口，到目前为止还是局限在自动化运维、网络优化调优上。而NFV作为SDN引发的设备重构还是局限在单点设备上。OpenConfig直击BGP要害，是真正对运营商网络、互联网基础IP网络进行开放、可编程大动作的第一步，是基于改良渐变走向公网大网控转分离、集中控制的第一步，是以退（抽象的厂商无关的BGP配置和策略模型）为进（改良渐变实现大网开放、可编程、集中控制）的巨大一步。

>DC领域白盒交换机已经非常清晰，而运营商十分期待的公网白牌化的路径还非常模糊，NFV也许是急先锋但只是网络中很小的一部分。Google基于自己实践牵头组建的Open Config工作组，这是打响公网白牌化的第一枪。

### 是否使用YANG的争论
OpenConfig从一开始就使用YANG语言。YANG已经诞生很长时间了并且通常用于传统网络建模，主要是通过命令行界面进行操作。YANG不够新颖，但是从OpenConfig的目标出发YANG是一个很好的建模框架。

```flow
st=>start: operators
ia=>operation: intent API
utm=>operation: update topology model
oym=>operation: OC YANG models
cg=>subroutine: configuration generation
cd=>operation: configuration data
dev=>operation: devices
e=>end

st->ia->utm->cg
oym(right)->cg(left)

```


## gRPC:多平台RPC框架
### gRPC特点
- 负载均衡、应用层流控制、调用取消
- 基于Protobuf序列化（高效的有线编码？）
- 多平台、支持多种语言
- 开源、开发者活跃

### gRPC扩展了HTTP /2的传输层
- 二进制成帧、头部压缩
- 双向流、支持服务器推送
- 跨请求和流的连接复用

## HTTP/2协议

HTTP/2的目标包括异步连接复用，头压缩和请求反馈管线化并保留与HTTP 1.1的完全语义兼容。
HTTP 2.0的首个草稿于2012年11月发布，其内容基本和SPDY协议相同。
HTTP/2主要以SPDY技术为主。SPDY（发音如英语：speedy），一种开放的网络传输协定，由Google开发，用来传送网页内容。基于传输控制协议（TCP）的应用层协议 。

HTTP/2主要针对HTTP/1.1在以下几个方面做的优化：

1. 并行能力有限
    每一个源最大只支持6个请求
    管道在实际使用时不起作用
    竞争性的TCP流，强制快速重传（Spurious retransmissions）
    额外的握手、内存缓冲等
2. 客户端请求队列
    队首阻塞
    延迟的请求分发
3. 较高的协议负载
    头信息和Cookies大约要800字节
    HTTP元数据没有压缩
4. HTTP/1.1只允许由客户端主动发起请求，服务端只能等待客户端发送请求

在HTTP/2中，基本的协议单位是帧，每个帧都有不同的类型和用途。例如，报头(HEADERS)和数据(DATA)帧组成了基本的HTTP请求和响应；其他帧，例如设置(SETTINGS)和推送承诺(PUSH_PROMISE)则用来实现HTTP/2的其他功能。

HTTP/2基于SPDY协议，充分解决了TCP连接的限制。它允许多个并发 HTTP 请求共用一个TCP会话，而不是为每个请求单独开放连接，这样只需建立一个 TCP连接就可以传送网页上所有资源，不仅可以减少消息交互往返的时间还可以避免创建新连接造成的延迟，使得 TCP 的效率更高。

针对只能由客户端发起请求的问题，HTTP/2添加了一种新的交互模式，即服务器能够通过复用一个以PUSH_PROMISE帧发送的请求来实现推送。

对于数据冗余问题，在HTTP/2中帧包含的HTTP报头字段是压缩的，同时它还舍弃掉了不必要的头信息，因此能显著地减少请求和响应的大小。

## SPDY协议
SPDY（发音如英语：speedy），一种开放的网络传输协定，由Google开发，用来传送网页内容。基于传输控制协议（TCP）的应用层协议。Google最早是在Chromium中提出的SPDY协议。目前已经被用于Google Chrome浏览器中来访问Google的SSL加密服务。SPDY并不是首字母缩略字，而仅仅是"speedy"的缩写。SPDY现为Google的商标。

SPDY当前并不是一个标准协议，但SPDY的开发组已经开始推动SPDY成为正式标准（现为互联网草案），HTTP/2主要以SPDY技术为主。Google Chrome，Mozilla Firefox，Opera和Internet Explorer均已支持SPDY协议。SPDY协议类似于HTTP，但旨在缩短网页的加载时间和提高安全性。SPDY协议通过压缩、多路复用和优先级来缩短加载时间。

### 设计
设计SPDY的目的在于降低网页的加载时间。通过优先级和多路复用，SPDY使得只需要建立一个TCP连接即可传送网页内容及图片等资源。SPDY中广泛应用了TLS加密，传输内容也均以gzip或DEFLATE格式压缩（与HTTP不同，HTTP的头部并不会被压缩）。另外，除了像HTTP的网页服务器被动的等待浏览器发起请求外，SPDY的网页服务器还可以主动推送内容。

### 与HTTP的关系
SPDY并不用于取代HTTP，它只是修改了HTTP的请求与应答在网络上传输的方式；这意味着只需增加一个SPDY传输层，现有的所有服务端应用均不用做任何修改。 当使用SPDY的方式传输，HTTP请求会被处理、标记简化和压缩。比如，每一个SPDY端点会持续跟踪每一个在之前的请求中已经发送的HTTP报文头部，从而避免重复发送还未改变的头部。而还未发送的报文的数据部分将在被压缩后被发送。








