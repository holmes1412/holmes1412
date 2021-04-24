# 10行代码实现C++服务器

我们可能曾经面临过，架构方案的频繁改动，是因为我们的项目需求总是随着时间推移而在不断迭代更新。可是，一个好的架构方案，应该是既可以满足项目初期对开发成本的需求，也可以支持项目中期丰富的功能，更可以满足后期对性能的吹毛求疵。

今天我们用10行代码写一个C++ Server，一起看看随着项目更迭，我们需求如何去解决不同的需求。

### 初期需求：开发快

我们现在有一个需求：实现一个最简单的服务器，收到请求之后给对方回复一句话就可以了。

如果是你，你会选择使用C++吗？很可能不会。因为使用python或者go，写一个server真的不要太简单，而C++不仅代码复杂，也没有语言级别的网络库，依赖一大堆东西想想就，好！头！疼！

但其实，用C++也可以像其他语言那样丝滑、清爽，简洁的10行代码足以实现一个server！我们以最最通用的国际标准协议——**Http**为例：

```cpp
int main() {
    WFHttpServer server([](WFHttpTask *task) {
        task->get_resp()->append_output_body("<html>Simplest C++ Server</html>");
    });
    if (server.start(8888) == 0) {
        getchar(); // press "Enter" to end.
        server.stop();
    }
    return 0;
}
```

这个server使用了**Workflow**，除了**SSL**以外，无需依赖其他第三方库，安装编译都非常简单。

- 帮你收发和解析Http协议
- 收到消息之后回复一个名字
- 在8888端口直接run起来

短短10行代码，一次满足你三个愿望。

你有没有想过，当我们在实现一个Server的时候，其实需要实现的是什么？

一个Server，与用户打交道的层面需要考虑的是：

1. 我可以接收和发出什么协议
2. 怎么表达一次网络交互
3. 怎么表达请求与回复
4. 我的业务逻辑该写在哪里？
5. 最后就是启动和退出。

伴随着以上这10行代码，我们详细地解读：

1. 我们选用**Http**协议，因此构造了一个``WFHttpServer``；
2. 一次网络交互就是一次任务，因为是**Http**协议，因此我们是``WFHttpTask``；
3. 对server来说，我的交互任务就是收到请求之后，填好回复，这些通过：``task->get_req()``和``task->get_resp()``可以获得；
4. 逻辑在一个函数中（即上面的lambda），表示收到消息之后要做的事情；
5. Server启动和退出使用``start()``和``stop()``两个简单的api，而中间要用``getchar();``卡住，是因为**Workflow**是个纯异步的框架。也许你现在用不到，但记住这个有用的功能，我们慢慢往后看～

### 中期需求：高吞吐

伴随我们史上最简单的C+ server代码顺利上线，业务迅速增长，我们开始对吞吐有要求了。

鉴于我们已经熟练使用C++开发server，我们可以来点实用点的场景：给用户做代理服务器。那么我们server的逻辑需要分三步：

1. 从用户的请求拿到想要访问的下游的地址；
2. 把用户的请求发给下游；
3. 把下游的回复填到我要给用户的回复中；

想象一样，如果我们收到请求之后在这个函数里忙等下游的网络请求，那么再有用户请求我的时候，我就没有线程去处理新用户了。

所以，**第一，我们希望server是多线程提供服务的。**

但再多的线程也可能会被霸占完的时候。我们需要无论server函数想要做任何耗时的操作，都不会影响到网络线程。

所以，**第二，我们希望网络线程和执行线程分离。**

如果服务只打算支持一万的QPS，其实底层怎么实现都很简单，但如果我们希望十万，甚至接近百万，则我们对server底层做收发的I/O模型有非常高的要求。

所以，**第三，以linux为例，我们希望对epoll的封装高效好用。**

带着以上高吞吐的需求，我们来看看**Workflow**内部的实现是怎样的：

// 此处有张图

基于以上的架构，基于**Workflow**的server轻轻松松就可以达到几十万QPS，高吞吐、低成本、开发快，简直是居家旅行开发代码必备的良药。

如果说**高吞吐**是对**整体请求**的性能要求，那么**低延迟**就是对**个体请求**的性能要求。高吞吐如果不进行单机优化，尚且可以通过堆机器解决；而低延迟就没有那么简单了。这是什么场景呢？我们继续往下看～

### 后期需求：低延迟

项目持续提供服务，用户反馈非常好。但现在的server性能越来越卷，能够高吞吐已经不是什么新鲜事了。这个时候，如果我们的server响应时间非常快，那么用户体验就会明显被提升一个量级，作为一个用户至上的团队，我们必须追求最极致的低延迟。

这个时候我们可以回顾第一部分提到的一个词：**纯异步**。

其实，我们理想中的server，无论是访问下游、还是进行文件的异步IO读取、乃至操作其他异步资源的，都应该是纯异步的。而低延迟的关键，就在于这些异步操作的调度彻底无损耗。

**Workflow**的代理服务器做纯转发的时候，性能是纯server的**1/2**，已是**理论最大值**，充分证明了Workflow是一个**调度无损耗**、**延迟超低**的**纯异步框架**。我们来看一下纯异步代理server应该如何实现呢？

第一步，我们来实现代理服务器的``process()``函数，这里我们可以接触到**Workflow**中常用的串行任务流``SeriesWork``的概念了：

```cpp
void process(WFHttpTask *proxy_task)
{
    auto *req = proxy_task->get_req(); // 1. 拿到用户当前的请求
    SeriesWork *series = series_of(proxy_task); // 2. 拿到我们代理服务器收到用户请求时所在的串行流series
    WFHttpTask *http_task; // 3. 创建一个请求下游的Http任务

    // 4. 整个异步代理流程中需要的上下文
    tutorial_series_context *context = new tutorial_series_context; 
    context->url = req->get_request_uri();
    context->proxy_task = proxy_task;

    // 5. 把上下文和串行流series的回调函数设置好
    series->set_context(context);
    ...

    // 6. 创建发往下游的http任务并把用户给我的内容填上去，并且填上http_callback回调函数
    http_task = WFTaskFactory::create_http_task(req->get_request_uri(), 0, 0,
                                                http_callback);
    req->set_request_uri(http_task->get_req()->get_request_uri());
    req->get_parsed_body(&body, &len);
    req->append_output_body_nocopy(body, len);
    *http_task->get_req() = std::move(*req);

    // 7. 把http任务串到当前series中，一切交给Workflow框架进行调度
    *series << http_task; 
}
```

这个时候，任务就不由我们管了。直到下游服务器给我们回复、并被Workflow框架收完这个网络消息之后，才会调起我们刚才填的那个``http_callbac()``回调函数。来看一下收到下游消息之后，我们需要做些什么吧：

```cpp
void http_callback(WFHttpTask *task)
{
    // 1. 当前的task为我们和下游服务器之间的网络任务
    int state = task->get_state();
    int error = task->get_error();
    auto *resp = task->get_resp();
    SeriesWork *series = series_of(task);

    // 2. 通过刚才的上下文，我们可以拿到一开始用户发给我们时的那个proxy_task了
    tutorial_series_context *context =
        (tutorial_series_context *)series->get_context();
    auto *proxy_resp = context->proxy_task->get_resp();
    ...
    //3. 把下游的回复填到我们给用户的网络任务的回复上
    *proxy_resp = std::move(*resp);
  }
```

最后，和我们初期写那10行server一样，无需任何回复的调用，并没有``task->reply()``接口，只要直接交给**Workflow**，而回复消息的时机是在``SeriesWork``里所有其它任务被执行完后，一切就可以自动回复、自动回收内存，满足程序员心中那高傲的小洁癖。

写到这里，我们已经可以使用C++来写任何一个**开发效率高**、**整体吞吐快**、**请求低延迟**的**纯异步server**了，妈妈再也不用担心我的架构方案无法满足项目长远的需求迭代了。但**Workflow**的功能远不止于此，而如果你对server的其他使用场景还有所好奇，那么欢迎点击[Worklfow](https://github.com/sogou/workflow)的github猎奇更多脑洞大开的功能和用法。
