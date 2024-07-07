# 《Workflow通用并发控制组件：ResourcePool资源池》

开源项目Workflow是C++异步调度的高性能框架，广泛用于高吞吐低延迟的网络服务器、并行计算和组装复杂网络请求的客户端等领域。在异步调度的编程范式下，想要实现并发控制是非常困难的，因为一旦无法做到无阻塞的调度，那么框架性能就会大打折扣。

线上非常常见的场景是：异步服务器需要限制用户的并发，从而保护有限的后端资源，并在超载时可以立刻拒绝用户或者实施排队等待的处理策略。

一个好的并发控制组件，应该是因框架制宜，实现上能够做到完全非阻塞，而语义上能做到足够简单、抽象、通用。因此在这里介绍一下C++ Workflow项目中的ResourcePool资源池，欢迎正在使用Workflow的小伙伴尝试，相信可以让你的代码简化不少，也欢迎有类似场景的开发者们参考与交流~~~

https://github.com/sogou/workflow

## 一、信号量

信号量（Semaphore）小伙伴们都知道，它由大名鼎鼎的Dijkstra发明（就是那个也发明与自己同名的最短路算法的计算机科学家）~ 这是一个用于并发和同步场景下的抽象概念：假设对任何对象我们都想要控制访问它的并发度为n，那么定义出两个简单的操作就可以做到。

- P操作：将n减1
- V操作：将n加1

结合定义，我们不难思考出这两个操作的底层逻辑：
1. 我们想操作任何一个资源的时候，就可以执行P；用完资源想还回去，可以执行V；
2. 资源不是无限的，执行P如果n减到要小于0了，说明资源不够，那你就等着；
3. 执行V把资源归还，意味着有人在等的时候，你可以叫醒这个人；

这个抽象的语义，具体到每个系统中的实现是不一样的。即使裸写C的小伙伴也很少会直接用操作系统的信号量，是因为我们都会用到更高级的封装。

虽然Workflow构思资源池的时候并不是奔着实现信号量的语义去做的，但是基于任务流的语义想要解决并发控制的问题时，会发现与信号量的概念殊途同归。

## 二、资源池

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


https://github.com/sogou/workflow/issues/1319 《如何根据用户配置去限制server的qps》
