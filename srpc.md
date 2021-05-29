# 用一个极简框架向你讲清楚RPC架构原理

**Remote Procedure Call**，是一个大家都不陌生的词，只要涉及到通信，必然需要某种网络协议。我们很可能用过HTTP，那么RPC又和HTTP有什么区别呢？RPC还有什么特点，常见的选型有哪些？今天我们用一个基于C++实现的开源项目SRPC，一文给你讲明白RPC的基本原理和层次架构。SRPC整体代码风格简洁、架构层次精巧，整体约1万行代码，非常适合用来学习RPC架构：https://github.com/sogou/srpc

### 1. RPC是什么

RPC可以分为两部分：用户调用接口 + 具体网络协议。前者为开发者需要关心的，后者由框架来实现。

举个例子，我们定义一个函数，我们希望函数如果输入为“Hello World”的话，输出给一个“OK”，那么这个函数是个本地调用。如果一个远程服务收到“Hello World”可以给我们返回一个“OK”，那么这是一个远程调用。我们会和服务约定好远程调用的函数名。因此，我们的用户接口就是：输入、输出、远程函数名，比如用SRPC开发的话，client端的代码会长这样：

```cpp
int main()
{
    Example::SRPCClient client(IP, PORT);
    EchoRequest req; // 用户自定义的请求结构
    EchoResponse resp; // 用户自定义的回复结构
  
    req.set_message("Hello World");
    client.Echo(&req, &resp, NULL); // 调用远程函数名为Echo
    return 0;
}
```

具体网络协议，是框架来实现的，把开发者要发出和接收的内容以某种应用层协议打包进行网络收发。这里可以和HTTP进行一个明显的对比：
- HTTP也是一种网络协议，但包的内容是固定的，必须是：请求行 + 请求头 + 请求体；
- RPC是一种自定义网络协议，由具体框架来定，比如SRPC里支持的RPC协议有：SRPC/thrift/BRPC/tRPC

这些RPC协议都和HTTP平行，是应用层协议。我们再进一步思考，HTTP只包含具体网络协议，也可以返回比如我们常见的HTTP/1.1 200 OK，但仿佛没有用户调用接口，这是为什么呢？

这里需要搞清楚，用户接口的功能是什么？最重要的功能有两个：

- 定位要调用的服务；
- 让我们的消息向前/向后兼容；

我们用一个表格来看一下HTTP和RPC分别是怎么解决的：

||定位要调用的服务|消息前后兼容|
|---|---|---|
|HTTP|URL|开发者自行在消息体里解决|
|RPC|指定Service和Method名|交给具体IDL|

因此，HTTP的调用减少了用户调用接口的函数，但是牺牲了一部分消息向前/向后兼容的自由度。但是，开发者可以根据自己的习惯进行技术选型，因为RPC和HTTP之间大部分都是协议互通的！是不是很神奇？接下来我们看一下RPC的层次架构，就可以明白为什么不同RPC框架之间、以及RPC和HTTP协议是如何做到互通的。

### 2. RPC有什么

我们可以从SRPC的架构层次上来看，RPC框架有哪些层，以及SRPC目前所横向支持的功能是什么：

* **用户代码**（client的发送函数/server的函数实现）
* **IDL序列化**（protobuf/thrift serialization）
* **数据组织** （protobuf/thrift/json）
* **压缩**（none/gzip/zlib/snappy/lz4）
* **协议** （Sogou-std/Baidu-std/Thrift-framed/TRPC）
* **通信** （TCP/HTTP）

我们先关注以下三个层级：

<img src="https://raw.githubusercontent.com/wiki/sogou/srpc/srpc-1.1.1.png" width = "719" height = "520" alt="srpc_protocol_transport_hierarchy" align=center />

如图从左到右，是用户接触得最多到最少的层次。IDL层会根据开发者定义的请求/回复结构进行代码生成，目前小伙伴们用得比较多的是protobuf和thrift，而刚才说到的用户接口和前后兼容问题，都是IDL层来解决的。SRPC对于这两个IDL的用户接口实现方式是：

- thrift：IDL纯手工解析，用户使用srpc是不需要链thrift的库的 ！！！
- protobuf：service的定义部分纯手工解析

中间那列是具体的网络协议，而各RPC能互通，就是因为大家实现了对方的“语言”，因此可以协议互通。

而RPC作为和HTTP并列的层次，第二列和第三列理论上是可以两两结合的，只需要第二列的具体RPC协议在发送时，把HTTP相关的内容进行特化，不要按照自己的协议去发，而按照HTTP需要的形式去发，就可以实现RPC与HTTP互通。

### 3. RPC的生命周期

到此我们可以通过SRPC看一下，把request通过method发送出去并处理response再回来的整件事情是怎么做的：

<img src="https://raw.githubusercontent.com/wiki/sogou/srpc/srpc-2.png" width = "900" height = "520" alt="srpc_one_round" align=center />    
       
根据上图，

可以更清楚地看到刚才提及的各个层级，其中压缩层、序列化层、协议层其实是互相解耦打通的，在SRPC代码上实现得非常统一，横向增加任何一种压缩算法或IDL或协议都不需要也不应该改动现有的代码，才是一个精美的架构～

我们一直在说生成代码，到底有什么用呢？图中可以得知，生成代码是衔接用户调用接口和框架代码的桥梁，这里以一个最简单的protobuf自定义协议为例：

```cpp
message EchoRequest
{
    optional string message = 1;
};

message EchoResponse
{
    optional string message = 1;
};

service ExamplePB
{
    rpc Echo(EchoRequest) returns (EchoResponse);
};
```
我们定义好了请求、回复、远程服务的函数名，接下来我们看看通过SRPC会生成出什么样的生成代码：

```cpp
// SERVER代码
class Service : public srpc::RPCService
{
public:
    // 用户需要自行派生实现这个函数，与刚才pb生成的是对应的
    virtual void Echo(EchoRequest *request, EchoResponse *response,
                      srpc::RPCContext *ctx) = 0;
};

// CLIENT代码
using EchoDone = std::function<void (EchoResponse *, srpc::RPCContext *)>;

class SRPCClient : public srpc::SRPCClient 
{
public:
    // 异步接口
    void Echo(const EchoRequest *req, EchoDone done);
    // 同步接口
    void Echo(const EchoRequest *req, EchoResponse *resp, srpc::RPCSyncContext *sync_ctx);
    // 半同步接口
    WFFuture<std::pair<EchoResponse, srpc::RPCSyncContext>> async_Echo(const EchoRequest *req);
};
```

作为一个高性能RPC框架，SRPC生成的client代码中包括了：同步、半同步、异步接口，文章开头展示的是一个同步接口的做法。

而server的接口就更简单了，作为一个服务端，我们要做的就是收到请求->处理逻辑->返回回复，而这个时候，框架已经把刚才提到的网络收发、解压缩、反序列化等都给做好了。一个完整的server示例如下

```cpp
#include "xxxx.srpc.h" // include生成代码头文件

class ExamplePBServiceImpl : public ::example::ExamplePB::Service
{
public:
    void Echo(::example::EchoRequest *request, ::example::EchoResponse *response,
              srpc::RPCContext *ctx) override
    { /* 收到请求后要做的事情，并填好回复;*/ }  
};

int main()
{
    // 1. 定义一个server
    SRPCServer server;

    // 2. 定义一个service，并加到server中
    ExamplePBServiceImpl examplepb_impl;
    server.add_service(&examplepb_impl);

    // 3. 把server启动起来
    server.start(PORT);
    pause();
    server.stop();
    return 0;
}
```

通过这篇文章，相信我们可以清晰地了解到RPC是什么接口长什么样，也可以通过与HTTP协议互通来理解协议层次，更重要的是可以知道具体纵向的每个层次及横向对比我们常见的每种使用模式都有哪些。如果小伙伴对更多功能感兴趣，也可以通过阅读SRPC源码进行进一步了解。

