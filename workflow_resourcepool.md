# 《Workflow通用并发控制组件：ResourcePool资源池》

开源项目Workflow是C++异步调度的高性能框架，广泛用于高吞吐低延迟的网络服务器、并行计算和组装复杂网络请求的客户端等领域。在异步调度的编程范式下，想要实现并发控制是非常困难的，因为一旦无法做到无阻塞的调度，那么框架性能就会大打折扣。

线上非常常见的场景是：异步服务器需要限制用户的并发，从而保护有限的后端资源比如GPU计算，并在超载时可以立刻拒绝用户或者实施排队等待的处理策略。

一个好的并发控制组件，应该是因框架制宜，实现上能够做到完全非阻塞，而语义上能做到足够简单、抽象、通用。因此在这里介绍一下C++ Workflow项目中的ResourcePool资源池，欢迎正在使用Workflow的小伙伴尝试，相信可以让你的代码简化不少，也欢迎有类似场景的开发者们参考与交流~~~

https://github.com/sogou/workflow

## 一、信号量Semaphore

信号量（Semaphore）小伙伴们都知道，它由大名鼎鼎的Dijkstra发明（就是那个也发明与自己同名的最短路算法的计算机科学家）~ 这是一个用于并发和同步场景下的抽象概念：假设对任何对象我们都想要控制访问它的并发度为n，那么定义出两个简单的操作就可以做到。

- P操作：将n减1
- V操作：将n加1

结合定义，我们不难思考出这两个操作的底层逻辑：
1. 我们想操作任何一个资源的时候，就可以执行P；用完资源想还回去，可以执行V；
2. 资源不是无限的，执行P如果n减到要小于0了，说明资源不够，那你就等着；
3. 执行V把资源归还，意味着有人在等的时候，你可以叫醒这个人；

这个抽象的语义，具体到每个系统中的实现是不一样的。

虽然Workflow构思资源池的时候并不是奔着实现信号量的语义去做的，但是基于任务流的语义想要解决并发控制的问题时，会发现与信号量的概念殊途同归。

![How WFResourcePool Works](https://raw.githubusercontent.com/wiki/holmes1412/holmes1412/workflow_resource_pool_1.1.png)

## 二、资源池ResourcePool

了解了语义和要解决的问题，那么我们看看资源池的接口，进一步拥有更具体的理解：

```cpp
class WFResourcePool
{
public:
    WFConditional *get(SubTask *task, void **resbuf);
    WFConditional *get(SubTask *task);
    void post(void *res);
    ...

protected:
    virtual void *pop()
    {
        return this->data.res[this->data.index++];
    }

    virtual void push(void *res)
    {
        this->data.res[--this->data.index] = res;
    }
    ...

public:
    WFResourcePool(void *const *res, size_t n);
    WFResourcePool(size_t n);
    ...
};
```

上述代码个人认为只需要划三个重点：构造函数、get()和post()。

**构造函数**有两个可选，简单版可以只传一个n。如果对于资源池有传递资源的需求，比如不知，那么还可以传入一个长度为n资源数组，数组每个元素为一个void *，内部会再分配一份相同大小的内存，把数组复制走。

因此相对上面两种情况，**get()函数**也有两个。get()等价于上述的P操作，用于拿一个访问资源的资格，如果使用第二个接口，那么是可以通过一个void **resbuf获得具体的资源。

**post()函数**用于资源使用完毕想归还的时候，也就是V操作，这里post()的res参数无需与get()得到res的一致。

## 三、与任务流结合：WFConditional

资源池的实现本身是很简单的，巧妙之处是它如何与Workflow当前的异步任务结合。答案就在上述get()函数的返回值，我们需要引入一层条件任务：WFConditional。

先回顾一下，Workflow中一个异步任务可以这样原地发射：

```cpp
auto *task = WFTaskFactory::create_xxx_task(x, x, callback);
task->start();
```

也可以扔到一个任务流中，待前序逻辑完成后执行，以实现任务编排和调度：

```cpp
SeriesWork *series; // 假设这一个现成的任务流。一般来自于我们自己创建，或者其他任务的callback中拿到
series->push_back(task);
```
在Workflow的实现中，它们的调度都不会卡住任何线程。

那么，现在加了一个资源池的约束，也就意味着：一个任务的发起，应当是'**当前流程允许执行**'并且'**从资源池中拿到资格**'才能真正得到调度。

**WFConditional条件任务**就是这个中间层：我们通过get()接口，把任务是否能通过资源池获得资格这个工作，交给了WFConditional，而WFConditional替代了这个任务本身被start或者放到任务流中。

![How WFConditional Takes Over Asynchronous Tasks](https://raw.githubusercontent.com/wiki/holmes1412/holmes1412/workflow_resource_pool_2.png)

用一个并发为1的小spider，展示一下常用的使用方式：

```cpp
WFResourcePool pool(1); // 0. 构造资源池，表示并发只允许1

// 1. 构造一个http任务。在它的callback中，我们把资源池的资源归还
WFHttpTask *t = WFTaskFactory::create_http_task(..., [](void *){pool.post(nullptr);});

// 2. 构造一个条件任务，它会在真正dispatch的时候才去尝试获取资源
WFConditional *c = pool.get(t, &t->user_data);  // 用user_data来保存res是一种实用方法。

// 3. 接下来用这个条件任务，替代http任务去和任务流结合
c->start(); // or series->push_back(c);
```
## 四、更多...

回到刚开始的场景，可以参考这个issue，这个也是内部用户咨询得最多的需求之一：《如何根据用户配置去限制server的qps》，🔗 https://github.com/sogou/workflow/issues/1319

当然我们实现一个server的时候，实际上是要控制用户请求的并发而非QPS，因为QPS是由资源处理能力附加了时间维度而得出的，核心指标应该是同时能处理的并发数。

细心的小伙伴还会发现，资源池有两个protected的接口，这是供聪明的你派生使用的。比如上述的post要叫醒最早在排队的那个人以实现先来先服务、还是叫醒最晚等待的那个人来减少超时用户的数量，这都是可以通过派生资源池来实现，而资源池本身只是一个非常通用的组件。

最近学习其他领域的知识也越来越发现，计算机中的语义无论是在操作系统还是框架还是算法层实现，都是相通的。也就是说如果对计算机底层逻辑能做到融会贯通，那么无论投身哪个细分领域都可以有新发现新贡献。虽然博主已经堕落到了Q更，最近才有空写一下资源池的介绍，但其实WFResourcePool的推出已经有2年了，本人单方面宣布它有可能是Workflow在开源之后才增加的组件中最重要的一个。非常感谢开发者们的支持，期待大家对更多场景的交流和共建，一起打造更多有趣的组件，发现更多通用的方案~
