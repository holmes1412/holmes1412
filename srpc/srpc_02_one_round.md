# 《一次RPC请求的全过程》

> 最近有计划给SRPC项目写几篇文章，协助小伙伴借助这个轻量级的框架快速了解RPC相关内容。   
> 本篇为第二篇，注重于解读一次请求在SRPC框架中的过程，是最简单、最主干的部分，而里边每一个层级怎么做资源调度和复用都不会包括在内，因此有基础的小伙伴可以直接跳读源码解析的那些部分。

## 1. RPC概念简述

花一点点时间补充RPC的基本概念，其字面意思是Remote Procedure Call，远程过程调用。也就是说：   
- 要么，我们是客户端，我们需要把某些事情交给远程机器去做；   
- 要么，我们是服务端，别人交给我一些事情让我做；   

那么必然涉及到三个小问题：   
1. 请求是什么？
2. 怎么指定调用哪个过程？
3. 怎么给填好回复？

而客户端就是发起的人；服务端就是启动服务器之后，等待他人发起。   

我们举个小例子，上述三个小问题可以这样：   
1. 请求是int a和int b；
2. 指定调用对方的sum；
3. 回复是求和后的值int ret；

一会儿用这个例子看看RPC框架的代码是怎么做的。   

## 2. 协议与框架

我们常提到的RPC可能是一种`框架`:用来帮我们做网络收发。   
比如SRPC是个轻量级的RPC框架，还有大家熟知的GRPC、Thrift等。

也可能是一种`协议`：RPC协议让不同语言、不同框架都可以互通。   
个人理解协议的本质是为了生态服务的，RPC承担的是衔接整个生态系统的桥梁。

两者关系：   
以SRPC为例，支持多种协议，包括**SRPC**、**Thrift**、**BRPC**、**tRPC**，也是目前`唯一的tRPC协议开源实现`，另外还可以收发**HTTP协议**。

我们给出一张RPC请求过程的图及其中涉及到的关键函数接口，然后正式开始下面的学习。

<img width="910" alt="image" src="https://user-images.githubusercontent.com/1880011/230064917-60ca3d6b-510c-4eec-9f81-bdbc78d07285.png">

这里我们看到几个有意思的事情：

1. 请求/回复，是对称的。
- 对Client来说，请求时发出SRPCRequest，收到SRPCResponse；
- 对于Server来说，收到SRPCRequest，回复时发出SRPCResponse；

2. 收/发，接口是对称的。
- 发消息的接口都是encode()，无论我要发的是SRPCRequest还是SRPCResponse；
- 收消息的接口都是append()，无论我要收的是SRPCRequest还是SRPCResponse；

3. client/server，也是对称的。
- client主动发出请求，回复时是被动调起callback()的（哪怕我们用同步接口，那也是调用完代码再往下走）；
- server被动接收请求，回复是process()处理完之后主动进行的。

## 3. 定义RPC接口

我们刚才三个小问题怎么定义呢？可以使用`protobuf`作为接口描述文件：

```proto
syntax="proto2";

message Request { // request包含了a和b
    required int32 a = 1;
    required int32 b = 2;
};
                                                                                   
message Response { // response包含了ret
    required int32 ret = 1;
};

service MyService { // 服务名，用来区分我们的服务
    rpc Sum(Request) returns (Response); // 调用名字为Sum的函数
};
```

也可以配合srpc小工具的api命令，产生一个简单的protobuf描述文件，并进行修改。命令参考如下：

```
./srpc api MyService
```
然后就可以根据提示，打开MyService.proto并编辑其中的接口定义。

## 4. step-0 : client发出请求

如总图的step-0，我们想要发出请求，就需要调用上述定义的RPC接口`Sum( )`：

~~~cpp
int main()
{                                                    
    MyService::SRPCClient client("127.0.0.1", 1412);                                                                                           
                                                                                                                                      
    Request req;                                                               
    Response resp;                                                             
    RPCSyncContext ctx;                                                            
                                                                                   
    req.set_a(1);                            
    req.set_b(2);
    client.Sum(&req, &resp, &ctx);
    ...
}
~~~

当然想要框架知道怎么从上述的protobuf文件进行调用，我们需要一些代码生成工作。这不是本篇的重点，因此这里仅列出一些命令供大家运行起来。   

我们根据刚才srpc小工具的示例，通过改好的proto文件把项目生成出来：

```
./srpc rpc my_rpc_project -f MyService.proto -p ./
```

我们打开生成代码`MyService.srpc.h`，可以看到刚才调用的`Sum()`函数的异步接口和同步接口，定义如下：

```cpp
class SRPCClient : public srpc::SRPCClient                                         
{                                                                                  
public:
    void Sum(const Request *req, SumDone done);
    void Sum(const Request *req, Response *resp, srpc::RPCSyncContext *sync_ctx);
    ...
};
```

## 5. step-1：框架处理请求

以上，我们作为RPC的用户，代码就告一段多了。接下来交给RPC框架干活，它要做的事情包括但不仅限于：

**把这个请求内容、以及用户要调用哪个服务（service）的哪个函数（method）告诉远程，并通过网络发送出去。**

我们想要了解一个框架如何工作时，首先要了解它是基于什么构建起来的，包括什么语言什么底层网络收发库等。  

SRPC是基于Workflow的任务流编程范式开发的，并使用了其携带的网络收发，因此我们可以不用手写I/O多路复用等事情，但是开发需要遵循Workflow的编程规范，即：`任务流`。

我们可以认为对于网络任务来说，一次会话就是一个task，对于client我们的task职责就在于把Request发给对方，收回Response。

还是`MyService.srpc.h`这个文件，我们看一下最简单的异步接口：

```cpp
inline void SRPCClient::Sum(const Request *req, SumDone done)                   
{                                                                               
    auto *task = this->create_rpc_client_task("Sum", std::move(done));          
                                                                                
    task->serialize_input(req);                                                 
    task->start();                                                              
} 
```

内部会构造出一个`RPCClientTask`，它被`task->start();`之后，就可以认为请求交给框架，用户态无需再关心，直到回复时框架通过回调等机制叫醒用户代码。

`RPCClientTask`的定义在SRPC框架代码里的`src/rpc_task.inl`，由于它比较复杂，我们挑重点看：

```cpp
// 1. 它派生于Workflow的WFComplexClientTask，
//    以REQ，RESP为模版，定义了内部请求与回复的格式，
//    我们这里分别是SRPCRequest和SRPCResponse。
template<class RPCREQ, class RPCRESP>                                              
class RPCClientTask : public WFComplexClientTask<RPCREQ, RPCRESP>
{
    ...
protected:
    // 2. SPRC框架重新实现了父类的方法message_out()，
    //    用来告诉Workflow网络层面这次发出的请求内容时啥
    CommMessageOut *message_out() override;

    // 3. 保存了一个rpc_callback, 让网络回复了之后通知SRPC框架
    //    SRPC框架再去做网络请求到用户Response的格式转换
    void rpc_callback(WFNetworkTask<RPCREQ, RPCRESP> *task);
};
```

上述的`RPCREQ`就是我们发出的请求，可以打开`src/message/rpc_message_srpc.h`。

`SRPCRequest`与`SRPCResponse`都从`SRPCMessage`派生：

```cpp
class SRPCRequest : public SRPCMessage
{
    ...
};

class SRPCResponse : public SRPCMessage
{
    ...
};
```

那么谁定义了 `SRPCMessage`的内存结构呢？就是`SRPC协议`。下图可以清晰地看到，我们在SRPC协议头部就有meta部分，上述提到的`service`和`method`就是填在里边。而后面的message就是我们的Request。

<img src="https://raw.githubusercontent.com/wiki/sogou/srpc/srpc-protocol.jpg" alt="srpc_protocol" align=center />

我们把消息按照上述结构，通过`SRPCMessage::encode()`接口填好。这是Workflow的接口，它会在进行网络发送时`entry->session->out->encode()`被调用。

```cpp
inline int SRPCMessage::encode(struct iovec vectors[], int max, size_t size_limit)
{
    // 这里用上了第二部分讲的：RPC协议。我们按照协议结构填内容。
}
```

## 6. step-2：与操作系统相关的网络操作

这部分在Workflow中实现，涉及到的网络基础知识很多，后续会针对性展开写学习心得，包括：

- 命名服务
- 目标选取
- 负载均衡
- 连接管理
- IO多路复用

等等，现在暂时跳过。

## 7. server接收请求

我们切换一下视角，来到上述总图的右半边，server要接收请求了。

当然server作为一个被动接收者，它需要先被用户启动起来。以下是用户代码：

```cpp
int main()                                                                         
{
    SRPCServer server; // 1. 构造一个server，负责网络请求

    MyServiceServiceImpl impl; // 2. 构造一个服务，负责实现Sum
    server.add_service(&impl);  // 3. 把服务实现加到server中
                                                           
    if (server.start(1412) == 0)  // 4. 传入端口，把server跑起来
    {
        printf("my_rpc_project SRPC server started, port %u\n", 1412);
        wait_group.wait(); // 5. server start也是异步的，暂时要卡住主线程不退出
        server.stop();
    }
    else                                                                           
        perror("server start");                                                    
                                
    return 0;                                                                      
} 
```

然后就可以愉快地按照SRPC协议来接受请求了。这是谁来做的呢？RPCServer来做的。


## 8. step-4：框架为server接受请求

我们打开`src/rpc_server.h`看看：

```cpp
// 1. 从Workflow的WFServer派生
//    由RPCTYPE::REQ和RPCTYPE::RESP来指定请求与回复的类型
template<class RPCTYPE>
class RPCServer : public WFServer<typename RPCTYPE::REQ,
                                  typename RPCTYPE::RESP>
{
…
protected:
    //  2. 需要实现怎么构造一次会话，即RPCServerTask
    CommSession *new_session(long long seq, CommConnection *conn) override;
    // 3. 调用具体server接口的地方
    void server_process(NETWORKTASK *task) const;
    ...
};
```

我们的父类WFServer是可以帮我们按照某种协议收网络包的，只需要：
1. 我们实现new_session()， new 一个RPCServerTask给它；
2. 在模版参中指定的RPCTYPE::REQ上实现append()接口，指引Workflow网络层面如何从操作系统收到的数据上切一份完整的REQ下来。

其中第一步不是必须的，但我们SRPC框架需要，因为我们在本次会话有一些上下文要处理。但现在我们只需关心REQ。

这个REQ就是刚才提到的SRPCMessage，打开`src/message/rpc_message_srpc.cc`看看它的append()接口：

```cpp
int SRPCMessage::append(const void *buf, size_t *size, size_t size_limit)
{
    ...
}
```

Workflow会不停掉用这里实现的append()来把SRPC协议图里的消息收完。

我们这里通过返回值来告知Workflow的网络层本条消息的接收情况：
1：消息接受完成；
0：未完成，继续收；
< 0：错误；

只要返回1，流程就会继续往下走，也就是到了process函数。


## 9. step-5：调用开发者的rpc函数

SRPC框架收完消息之后，需要对meta进行一些处理：
根据meta里的service去找到用户刚才server.add_service(impl)时的那个service；
根据meta里的method去找用户的impl里实现的函数；

然后就可以调用server端开发者实现的rpc函数了。

查找过程很简单，框架源码在`src/rpc_server.h`，以下是简化的流程：

```cpp
template<class RPCTYPE>                                                            
void RPCServer<RPCTYPE>::server_process(NETWORKTASK *task) const
{
    // 1. 把SRPC协议中的meta信息反序列化出来
    req->deserialize_meta(); 
    ...
    // 2. 找service
    auto *service = this->find_service(req->get_service_name());
    ...
    // 3. 找method
    auto *rpc = service->find_method(req->get_method_name());
    ...
    // 4. 进一步处理
    status_code = (*rpc)(server_task->worker);
    ...
}
```

注意上述的进一步处理是因为，我们还需要对body进行反序列化，具体代码在`src/rpc_service.h`

```cpp
template<class INPUT, class OUTPUT, class SERVICE>                                 
static inline int                                                                  
ServiceRPCCallImpl(SERVICE *service,                                               
                   RPCWorker& worker,                                              
                   void (SERVICE::*rpc)(INPUT *, OUTPUT *, RPCContext *))          
{
    // 1. new 一片请求，也就是一开始在protobuf中定义的包含a和b的Request
    auto *in = new INPUT;

    // 2. 按照网络包里的body，从req反序列化处出来到in上
    int status_code = worker.req->deserialize(in);

    // 3. new 一片回复，也就是一开始在protobuf中定义的包含ret的Response
    auto *out = new OUTPUT;

    // 4. 调用用户代码实现的rpc函数，进行计算
    (service->*rpc)(in, out, worker.ctx);
}
```

之后就可以交给框架返回了。

## 10. 对称的回程

我们最后简单看一下用户代码里一般长啥样，也就是刚才提到的server_main.cc中的impl里的rpc:

```cpp
class MyServiceServiceImpl : public MyService::Service                          
{                                                                               
public:
    void Sum(Request *request, Response *response, srpc::RPCContext *ctx) override
    {
        // 这里是我们自己实现的加法
        response->set_ret(request->a() + request->b());                                         
    }                                                                           
}; 
```

之后，用户无需进行任何代码编写，SRPC和Workflow会进行step-6和step-7，与先前的步骤类似且对称地，把回复填好并发出。

而client端又会先从Workflow和SRPC进行step-8和step-9，同样与上述步骤类型且对称地，把回复收好，并调用到我们的callback，或者在同步接口中（也就是文中的Sum调用示例）填好Response，此次请求就完整结束了。

```cpp
int main()
{
    ... 
    client.Sum(&req, &resp, &ctx);
                                                             
    if (ctx.success)                                                               
        fprintf(stderr, "ret = %d\n", resp.ret());            
    else                                                                           
        fprintf(stderr, "sync status[%d] error[%d] errmsg:%s\n", ctx.status_code, ctx.error, ctx.errmsg.c_str()); 

    return 0;
}

附上从调用模块角度来看的one round图：

<img src="https://raw.githubusercontent.com/wiki/sogou/srpc/srpc-2.png" alt="srpc_one_round" align=center />  

更多内容参考：https://github.com/sogou/srpc/blob/master/docs/wiki.md
