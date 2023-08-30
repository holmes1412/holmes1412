# srpc系列4: 基于Workflow编写复合任务(1)

> 周末两天刚好上课，遂有空给SRPC系列写最复杂的模块rpc_task。
> 通过本篇可以了解SRPC任务调度的核心，同时也是一个基于Workflow复合任务做派生非常典型的实现案例，适合想基于Workflow做二次开发的小伙伴。 
> 为了限制篇幅，这篇只来得及解读client了~~~ 
> 整体阅读预计8分钟

[https://github.com/sogou/srpc](https://github.com/sogou/srpc)

【SRPC派生Workflow复合任务，多个虚函数的实现与调用流程图】

## 0. 基本原理

由于鸽了比较久，和大家一起复习一下，我个人理解的Workflow任务流核心的核心概念：

- 我们知道Workflow中的任何一个任务，都从被调用dispatch()开始、从被调用done()结束
- dispatch()通常代表着我们主动发起，done()通常代表着系统异步资源结束时被动发起
- done()调用结束后再从任务流中拿出+发起下一个任务，任务就可以生生不息地执行下去

这就是Workflow使用任务流实现结构化并发的基本原理。

（以上非官方理解，如何有了偏差，希望大家去GitHub翻一下源码呜呜呜呜QAQ～

 [https://github.com/sogou/workflow](https://github.com/sogou/workflow)

那么，有了基本的任务，我们就可以通过一些固定的模式进行**任务组装**。

模式表示一些固定的逻辑，而具体的异步资源交给对应的基础任务来负责，无需耦合到模式中，从而做到分工明确，层次清晰。

如果说基础任务是代码块，那么模式就仿佛是一些循环和分支。

有了这些，我们就可以实现并行排序，或者带重试的任务。当然更复杂的，就是我们的Client网络请求了：

【HTTP复合任务图】

## 1. WFComplexClientTask

**Workflow**中提供了client复合任务基础类`WFComplexClientTask`，可以覆盖最基本的client网络需求：
- 向具体目标发起网络请求(REQ)；
- 收到网络回复(RESP)后调用回调函数；
- 重试；
- 重定向；
- 内置命名服务（比如DNS及更复杂的命名服务）；

我们经常使用的上层任务，包括**HTTP**、**Redis**、**MySQL**、**Kafka**等，都是从这个类派生的。

我们可以从下面这张图里找到它：

【complex派生图+圈】


但是不派生，也足够我们做基本的网络需求了，比如[自定义协议的tutorial](https://github.com/sogou/workflow/blob/master/docs/tutorial-10-user_defined_protocol.md)中，我们写好协议的实现，作为模版构造**WFComplexClientTask**直接使用即可。

## 2. 什么情况下需要派生？

但如果，我们希望基于上述的模式做一些策略调整，比如不一样的路由方式、认证方式、或者做一些埋点工作，那就需要派生了。

接口可以简单通过定义查看：[src/factory/WFTaskFactory.inl](https://github.com/sogou/workflow/blob/master/src/factory/WFTaskFactory.inl#L68)

```cpp
template<class REQ, class RESP, typename CTX = bool>                            
class WFComplexClientTask : public WFClientTask<REQ, RESP>
{
protected:
    using task_callback_t = std::function<void (WFNetworkTask<REQ, RESP> *)>; 

public:
    WFComplexClientTask(int retry_max, task_callback_t&& cb);

protected:
    // init()阶段调用，也就是构造任务的时候
    virtual bool init_success() { return true; }
    virtual void init_failed() {}

    // dispatch()阶段调用
    virtual bool check_request() { return true; }
    virtual WFRouterTask *route();

    // done()阶段调用
    virtual bool finish_once() { return true; }

public:
    ...
};
```
上面可以清晰看到，留给派生类的虚函数分三种，分别是**create**任务时、**dispatch**任务时和**done**任务时调用的。

注意一个任务是可以被dispatch和done多次的，只要这个任务没做完，就可以继续把它放回任务流。一个很简单的例子，重试这件事可以让一个任务执行多次，而无需反复创建和销毁任务。

## 2.RPCClientTask用到的虚接口

上述只是目前派生类所需要的功能的全集，一般来说无需都实现，而RPCClientTask正好是一个相对简单的实现，因此借助RPC的实现来具体理解一下这些阶段与虚接口。

[src/rpc_task.inl](https://github.com/sogou/srpc/blob/master/src/rpc_task.inl#L107)

```cpp
class RPCClientTask : public WFComplexClientTask<RPCREQ, RPCRESP>
{
protected:
    void init_failed() override;   // 从WFComplexClientTask派生
    bool check_request() override; // 从WFComplexClientTask派生
    CommMessageOut *message_out() override;
    bool finish_once() override;   // 从WFComplexClientTask派生
    void rpc_callback(WFNetworkTask<RPCREQ, RPCRESP> *task);
    int first_timeout() override { return watch_timeout_; }
public:
    ...
};
```

上述实现了三个虚接口，刚好分别是复合client任务的三个阶段：

由于步骤拆得比较碎，因此整体我们会使用task上的状态码来标记整个流程运行状态。

### 2.1 init阶段：init_failed()

`init_success()`和`init_failed()`都是init阶段被调用的。如果`init_failed()`被调用到，说明父类任务创建阶段初始化时失败了（比如用户的URI非法）。对于RPC任务来说，我们会把失败记录下来：

```cpp
template<class RPCREQ, class RPCRESP>
void RPCClientTask<RPCREQ, RPCRESP>::init_failed()
{
    init_failed_ = true; // init_failed_默认值为false
}
```

光看这里比较难以理解为什么要这么做。但其实RPC和基本网络协议不同，RPC涉及到IDL，所以在创建任务后、发起请求前，还有一个步骤就是**serialize_input()** : 把protobuf或者thrift按照IDL文件中的定义序列化到一片内存中。因此如果任务初始化失败，则不执行内部序列化。

// 所以同理，我们实现`init_success()`的话，就可以在初始化成功后做一些事情。

### 2.2 dispatch阶段：check_request()

dispatch阶段会调用`check_request()`。于是我们刚才**serialize_input()**的序列化结果，就可以在这个阶段通过返回值告诉父类，成功的话父类才会真正发起网络请求：

```cpp
template<class RPCREQ, class RPCRESP>
bool RPCClientTask<RPCREQ, RPCRESP>::check_request()
{
    int status_code = this->resp.get_status_code();
    return status_code == RPCStatusOK || status_code == RPCStatusUndefined;
}
```

### 2.3 done阶段：finish_once()

done阶段会调用`finish_once()`，顾名思义，每done()一次会调用一次。

RPC里做的事情就是把网络回复RESP的头部meta元数据进行反序列化，并设置好状态码。

```cpp
template<class RPCREQ, class RPCRESP>
bool RPCClientTask<RPCREQ, RPCRESP>::finish_once()
{
    int status_code = this->resp.get_status_code();

    if (this->state == WFT_STATE_SUCCESS &&
        (status_code == RPCStatusOK || status_code == RPCStatusUndefined))
    {
        if (this->resp.deserialize_meta() == false)
            this->resp.set_status_code(RPCStatusMetaError);
    }

    return true;
}
```

返回值是用于告诉父类，当前发出了的这个请求包，是否是用户的那个请求包

大家可能会觉得很奇怪，怎么不算呢？

这个我们放到后面结合 **message_out()** 接口再讲解～总之，当前都是返回true。

## 3. 还有哪些好用的虚接口？

刚才看到**RPCClientTas**k里，还有两个override的函数：

```cpp
{
    CommMessageOut *message_out() override;
    int first_timeout() override { return watch_timeout_; }
}
```

它们属于消息最底层的类：`CommSession`。

**message_out()**：用于告诉网络通信器要发出的message是什么，一般来说返回我们的REQ就可以了。需要派生实现的场景一般是：

- 有些阶段比如认证，我们希望需要发出的认证包和REQ不一样，则这个时候，就可以配合刚才提到的finish_once()返回值返回false，来让WFComplexClientTask父类知道当前任务还没完，不能从任务流中移除当前任务；
- 又或者做任何需要这个阶段做的一些事情；

**first_timeout()**：用于告诉网络通信器第一次请求的回复超时，一般来说是0，则会取常用的recv_timeout作为超时。需要派生的场景就很容易想象：

 - 比如watch一个东西是否有改变，则第一次超时可以设置更长；

`first_timeout()`在SRPC中就是用来返回用户设置的watch_timeout参数的，我们重点看看`message_out()`：

```cpp
template<class RPCREQ, class RPCRESP>
CommMessageOut *RPCClientTask<RPCREQ, RPCRESP>::message_out()
{
    this->req.set_seqid(this->get_task_seq());

    int status_code = this->req.compress(); // 1. 对请求进行压缩

    void *series_data = series_of(this)->get_specific(SRPC_MODULE_DATA);
    RPCModuleData *data = (RPCModuleData *)series_data;

    if (data) // 2. 埋点相关：把任务流上要透传的数据拿出
        this->set_module_data(*data);
    data = this->mutable_module_data();

    for (auto *module : modules_) // 2. 埋点相关：挨个调用要在这个埋点做处理的模块
        module->client_task_begin(this, *data);
    
    this->req.set_meta_module_data(*data); // 2. 埋点相关：把要透传的数据按需填到请求中

    if (status_code == RPCStatusOK)
    {
        if (!this->req.serialize_meta()) // 3. 序列化RPC协议里的meta
            status_code = RPCStatusMetaError;
    }

    if (status_code == RPCStatusOK)
        return this->WFClientTask<RPCREQ, RPCRESP>::message_out(); // 4. 交给父类发送

    this->disable_retry();
    this->resp.set_status_code(status_code);
    errno = EBADMSG;
    return NULL;
}
```

**Workflow**内部默认是没有嵌入任何埋点的，但我们可以很轻松按照上面的做法进行实现。这里的点是`用户请求发出之前`，我们还有许多地方可以埋点，比如**建立连接之前**（做认证）、**发出请求之前**（透传链路信息）、**收到请求之后**（打日志）等等。

这部分比较偏云原生，后续会单独开篇详述。

## 4. bind用户的回调

在第一部分还出现了一个函数，是`rpc_callback()`，它的实现和`message_out()`几乎是完全相反的，因此值得我们看一下。

RPCClient会把用户实际的回调函数bind到这里，因此任务结束后会回调到rpc_callback()，最后再交会给用户。

```cpp


template<class RPCREQ, class RPCRESP>                                           
void RPCClientTask<RPCREQ, RPCRESP>::rpc_callback(WFNetworkTask<RPCREQ, RPCRESP> *task)
{
    RPCWorker worker(new RPCContextImpl<RPCREQ, RPCRESP>(this, &module_data_),  
                     &this->req, &this->resp);

    int status_code = this->resp.get_status_code();

    if (status_code != RPCStatusOK && status_code != RPCStatusUndefined)
    {
        this->state = WFT_STATE_TASK_ERROR;
        this->error = status_code;
    }
    else if (this->state == WFT_STATE_SUCCESS)
    {
        status_code = this->resp.decompress(); // 1. 解压
        if (status_code == RPCStatusOK)
        {
            this->resp.set_status_code(RPCStatusOK);
            status_code = user_done_(status_code, worker); // 2. 反序列化meta
        }
    }
    ...

    if (!modules_.empty()) // 3. 埋点相关
    {
        void *series_data;
        RPCModuleData *resp_data = this->mutable_module_data(); // 3. 把透传数据拿出来
        ...

        for (auto *module : modules_)
            module->client_task_end(this, *resp_data); // 3. 每个模块按需处理相关数据
    }
    ...
}
```

整个流程到目前为止就非常清晰了，结合第一张图，相信小伙伴对client任务一次流程的理解会更加深刻。

希望明天的课也可以摸鱼，这样就可以把server也写完～
