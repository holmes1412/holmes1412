# Workflow的Redis协议网络交互详解

一个网络请求在发送之前，常常涉及多个前置的网络交互，如登陆、加密、获取元数据等等。Workflow希望让这些请求纯异步收发，并且不需要用户去操心这些前置请求，因此内部的**Http、Redis、MySQL、Kafka、Consul**等多个网络协议，都是基于任务流的编程范式重新实现的。

今天选择Redis协议：一个非常简洁优雅的数据向网络协议，给大家解读一下Workflow复合网络交互的巧妙设计，同时以Redis的订阅模式，作为Workflow支持streaming的实现案例。

https://github.com/sogou/workflow/

### 简洁的异步接口

用Workflow发出一个redis请求，用户侧只需要4步：

1. 填好地址
```cpp
const char *url = "redis://:password@host:port/dbnum?query#fragment";
```
workflow统一使用url来表达网络协议的目标，内部自带DNS、负载均衡等必要功能，我们只需要把地址、端口、密码等用户层信息填到url上。

2. 创建Redis任务
```cpp
WFRedisTask *task = WFTaskFactory::create_redis_task(url, RETRY_MAX, callback);
```
每种网络协议都有一个基础任务（比如WFRedisTask），而内部创建的是一个行为派生的复合任务（比如ComplexRedisTask），但对用户来说无需感知。

3. 设置请求命令和内容
```cpp
task->get_req()->set_request("SET", {"testkey", "testvalue"});
```
task上有REQ也有RESP，作为client我们拿出REQ，并设置请求命令。在这个请求处理完后，会把这个task通过上面的callback函数返回给我们，就可以把RESP拿出来用了。

4. 发起任务
```cpp
task→start();
```
这里可以用workflow中的任何任务流使用方式：比如直接`start()`；或者通过`push_back()`放到其他任务后面，以实现任务编排。

到这里，熟悉workflow的小伙伴可能觉得，仿佛什么都没讲呀？是的，用法上其他任务一样，没有任何区别。但其实，Workflow已经前置做了协议所要求的多次异步交互。

### 复合的网络交互

前置请求不一定是每次必须的：
- 比如登陆认证，每个连接认证一次就行，后面是连接复用的；
- 比如获取meta信息，全局获取一次就行，所有连接都可以共享这份数据；

上面的例子中的url有密码，也有选库，前者通过AUTH命令、后者通过SELECT命令与redis server做交互，因此我们需要在连接的第一个用户请求之前做完这些事情。

如何实现，可以有两种方式：

- 前面放一个新的任务，专门用来前置请求
- 还是使用同一个任务，先发出认证的REQ，但是这个任务会被拉起多次

// TODO： 和后面订阅呼应？前面提到，redis是一个简洁而美丽的协议，

// 这里有一张图

实现方式也很简单：task可以通过get_seq()拿到这是当前连接上的第几个请求。只要是第一个，那么就应该发送前置请求。

具体代码如下：

```cpp
CommMessageOut *ComplexRedisTask::message_out()
{
    long long seqid = this->get_seq();

    if (seqid <= 1)
    {   
        if (seqid == 0 && (!password_.empty() || !username_.empty()))
        {
            auto *auth_req = new RedisRequest;

            if (!username_.empty())
                auth_req->set_request("AUTH", {username_, password_});
            else
                auth_req->set_request("AUTH", {password_});

            succ_ = false;
            is_user_request_ = false;
            return auth_req;
        }

        if (db_num_ > 0 &&
            (seqid == 0 || !password_.empty() || !username_.empty()))
        {
            auto *select_req = new RedisRequest;
            char buf[32];

            sprintf(buf, "%d", db_num_);
            select_req->set_request("SELECT", {buf});

            succ_ = false;
            is_user_request_ = false;
            return select_req;
        }
    }   

    return this->WFComplexClientTask::message_out();
}
```

-

### streaming实现案例：订阅模式

除了上述一来一回的网络模式，Workflow也可以支持流式的协议，而Redis协议的订阅模式是一个典型的应用场景。

用法也很简单：

- 构造任务的时候指定要订阅的channel，内部会先发出一个订阅请求；

- 定义extract函数：每次收到消息的处理函数

- 定义callback函数：整个任务结束的回调函数，与基本任务无异；

// 图

具体如何实现呢？Workflow陆续为网络任务提供了两个新用法：

1、连续读数据：PackageWrapper 

一般来说，append函数用返回值表示0表示当前消息还没收完；返回值1表示已经收完，这时候会把kernel中连接的读状态删掉，并进入用户的回调，这就是消息一来一回的简单逻辑。

而PackageWrapper的append函数的实现是：当收完一条完整的消息之后，会把这条消息交给next_in()去使用。如果next_in()函数产生了一条新新的消息，那说明当前的streaming需要继续收，通过返回值为0告诉kernel当前继续读，从而实现streaming收消息。

2、连续写数据：WFNetworkTask::push()

这个接口用于同步写数据。由于同步写，因此无需经过kernel中的状态改变，这是workflow一直都支持的功能。


相关阅读（超长预警：https://zhuanlan.zhihu.com/p/172485495

基于以上功能，就可以实现流式的网络交互。

在redis的复合订阅任务ComplexRedisSubscribeTask中，就包含了一个Wrapper，它的next_in()函数的实现非常关键：根据redis协议处理，并在正常watch期间每次收到一个redis协议都会调用一次extract，从而实现上图的连续收数据的功能。

当然一看这个任务这个名字这么长，就知道单纯使用的小伙伴不需要关心~

最后，ref一下最近群里小伙伴问起的：“Workflow如何实现订阅功能？”
