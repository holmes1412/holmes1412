# 《 C++高并发异步定时器的实现 - Workflow架构系列 》 

对于C++高并发场景，定时问题处理起来有三大难题：高效、精准、原子性。除了定时任务随时可能**到期**、而进程随时可能要**退出**之外，最近Workflow甚至为定时任务增加了**取消**功能，导致任务可能被框架调起之前被用户取消，或者创建之后不想执行直接删除等情况，而这些情况大部分来说都是**由不同线程执行**的，因此其中的并发处理可谓教科书级别！

以下和大家一起看看Workflow在定时器的设计上做了哪些考虑，深扒细节，体验并发架构之美。

https://github.com/sogou/workflow

## 1. 高效的数据结构与timerfd

举个例子：实现一个server，收到请求之后，隔1s再回复给用户。

聪明的读者肯定知道，在server的执行函数中用**sleep(1)**是不行的，`sleep()`这个系统调用是会阻塞当前线程的，而异步编程里阻塞线程是高效的大忌！

所以我们可以使用[`timerfd`]((https://man7.org/linux/man-pages/man2/timerfd_create.2.html))，顾名思义就是用特定的fd来通知定时事件，把定时时机响应和网络事件响应都一起处理，用epoll管理就是一把梭。

现在离高效还差一丢丢。回到例子，我们不可能每次收到一个请求都创建一个timerfd，因为高并发场景下一个server通常要抗上百万的QPS，目前Workflow的超时算法做法是一个poller一个timerfd，内部利用了链表+红黑树的数据结构，时间复杂度在O(1)和O(logn)之间，其中n为poller线程的fd数量。

/* 并且超时处理目前看不是瓶颈所在，因为Linux内核epoll相关调用也是O(logn)时间复杂度，我们把超时都做到O(1)也区别不大。 */这个数据结构的组合还是比较有意思的，但我们先专注解决生效两个问题。

// wokrflow_timer_1.png

## 2. 精准的响应

于是上述定时问题就变成了：每收到一个请求就发起一个异步任务，在timeout之后可以通过callback通知我们，再作回复。

Workflow源码在kernel和factory中都有对应的实现，kernel层是主要负责timerfd的地方，当前factory层还比较薄。我们重点看看上述数据结构：`src/kernel/poller.c`。

由用户发起异步任务，将这个任务加到上述的链表+红黑树的数据结构中，如果这个超时是当前最小的过期时间，还会更新一下timerfd。

//////// 这里代码可以不用了 /////////
```cpp
int poller_add_timer(const struct timespec *value, void *context, poller_t *poller)                                          
{
    ...

    // 1. 分配内部用于poller内部管理的上下文
    node = (struct __poller_node *)malloc(sizeof (struct __poller_node));
    if (node)
    {
        // 2. 设置节点的值，对于定时任务，这个标志位【removed】后续会发挥关键作用
        node->data.operation = PD_OP_TIMER;
        node->removed = 0;
                                  
        // 3. 把超时设置上
        node->timeout.tv_sec += value->tv_sec;                                  
        node->timeout.tv_nsec += value->tv_nsec;

        // 4. 在锁内把节点加到上述数据结构中
        pthread_mutex_lock(&poller->mutex);                                     
        __poller_insert_node(node, poller);
        pthread_mutex_unlock(&poller->mutex);                                   
        return 0;                                                               
    }
    return -1;
} 
```
//////////////////

由框架的网络线程响应到超时，把数据结构中已经超时的全部节点都拿出来，调用任务的handle：

```cpp
/*** poller响应timerfd的到时事件，并处理所有到时的定时任务，当前在网络线程 ***/
static void __poller_handle_timeout(const struct __poller_node *time_node, poller_t *poller)                           
{                                                                               
    ...

    // 1. 锁里，把list与rbtree上时间已到的节点都从数据结构里删除，临时放到一个局部变量上                                        
    list_for_each_safe(pos, tmp, &poller->timeo_list)                           
    { ... }

    if (poller->tree_first)                                                     
    { ... }  

    // 2. 锁外，设置state和error，并回调Task的handle()函数
    while (!list_empty(&timeo_list))                                            
    {                                                                           
        node = list_entry(timeo_list.next, struct __poller_node, list);         
        list_del(&node->list);                                                  
                                                                                
        node->error = ETIMEDOUT;                                                
        node->state = PR_ST_ERROR;                                              
        free(node->res);                                                        
        poller->cb((struct poller_result *)node, poller->ctx);                  
    }                                                                           
} 
```

由于timerfd使用的超时时间是所有节点中最早超时的时间，而所有节点都在rbtree和list上按序排好，我们从前到后找的都是已超时的节点。因此利用了timerfd的精准性可以非常准确地叫醒当前已经超时的全部节点。由于实际使用中，用户使用的超时时长总是固定的，比如上述例子都是1s，因此超时的绝对时间一般来说都是递增的，这个数据结构的设计可以很好地满足定时器的需求。

## 3. 原子性

上述有提到，用户的回调需要调且只调一次，和**timeout**竞争的是**stop**，其实现逻辑与`poller_handle_timeout()`逻辑非常类似：用锁来互斥，抢到锁从相关数据结构里把节点拿出来，谁抢到谁调回调。

```cpp
/*** 进程退出，需要立刻打断所有定时任务 ***/
void poller_stop(poller_t *poller)
{
    ...
    poller->stopped = 1; // 1. 全局置标志位【stopped】，这个是重点，它是在锁外设置的

    // 2. 锁内，把rbtree里的每个节点拿下，统一在list中挨个处理
    list_for_each(pos, &node_list)
    {
        node = list_entry(pos, struct __poller_node, list);
        if (node->data.fd >= 0)     
        { ... }
        else
            node->removed = 1; // 标志位【removed】，下一部分会用到
    }

    ...
    // 3. 与timeout流程类似，挨个节点设置state和error，并回调
}
```

## 4.允许取消（新功能）

看到这里，会发现定时器的基本功能实现也不难，它甚至可以轻松地在进程退出时立刻全部到时结束。

但是当我们打开poller.h，感受一下它的接口，总觉得差了点什么：

// workflow_timer_2.png

是的！一个timer可以add，但是却不可以delete！

Workflow中许多结构的实现都是非常完备和对称的，因此**取消一个定时任务**这件事在Workflow开源的第一天就一直想要实现。

但是多加一个**取消cancel**会有很大的问题：如果一个timer已经timeout，用户再去cancel的**生命周期**是没有办法保证的，它可能已经**析构**了！

最近终于找到了一个非常好的解决办法：**使用命名timer，交给全局map管，cancel的时候带名字去操作，就可以解决生命周期问题**。

我们增加了带名字的Timer ：`__WFNamedTimerTask`，通过全局的map可以找到它，从而进行删除。删除实际上就是从poller中删除一个timer。所以从底向上先增加这个接口：

```cpp
/*** 取消一个定时任务时，从poller删除它 ***/
int poller_del_timer(void *timer, poller_t *poller)
{
    ...
    // 锁内：如果这个标志位还是0，表示stop还没把它拿走，这里就可以去删除这个timer
    if (!node->removed)
    {
        node->removed = 1; // 可以让cancel和stop互斥，保证只调一次

        if (node->in_rbtree) // 从定时器的数据结构中删掉
            __poller_tree_erase(node, poller);
        else
            list_del(&node->list);
        ...
    }
    ...

    // 锁外：标志位【stopped】可以保证cancel与stop调用回调的原子性：调且只调一次
    if (stopped)
        poller->cb((struct poller_result *)node, poller->ctx);

    return -!node;
}
```

刚才讲述过的**timeout**（时间到）、**stop**（进程退出）、**cancel**（用户取消）三者可能由三个线程分别发起，于是我们看到的并发状态，简单来说是这样的：

// wokrflow_timer_3.png
[现在的并发状态，timeout、stop、cancel三者可能随时发生]

## 5. 精妙的并发状态分析

cancel和另外两个行为有着本质上的不同：
- timeout和stop的触发顺序是先poller层被叫醒、再到Task层处理，最后调用用户的callback；
- cancel的触发是用户发起删除、Task层的命名map中先删掉它、再从poller里的数据结构中删掉，最后让Task的handle调用用户的callback；

因此先讨论第一类情况。上述流程中三个事件，假如**timeout**或者**stop**先发生。我们以**timeout**为例：

1. poller层面的`__poller_handle_timeout()`会把上述的**removed**标志位用上，与`poller_del_timer()`互斥：谁第一个抢到**removed**标志位并置为1，就代表了timer结束于哪个状态。如果是timeout，那么用户拿到的state为SS_STATE_COMPLETE，如果是stop则用户会拿到SS_STATE_DISRUPTED；
2. 互斥锁`poller->mutex`保证从poller的数据结构中删掉这个节点并调Task层回调，从而可以保证**stop**的时候无需重复处理它；
3. timeout调用的Task层回调，实际上是__WFNamedTimerTask::handle()：
(1)它会置一个标志位node_.task，表示此任务已经处理过；
(2)并且把这个节点从全局map中删除：这样就保证了用户不会**cancel**到它了；

```cpp
/*** 处理定时器到期，由poller调用 ***/
void __WFNamedTimerTask::handle(int state, int error)    
{    
    if (node_.task) // 由于不想先加锁再处理，所以先判断任务没有被cancel处理过
    {
        std::lock_guard<std::mutex> lock(__timer_map.mutex_);    
        if (node_.task) // 锁内再检查一下，入门技能
        {
            timers_->del(&node_, &__timer_map.root_); // 从map中删除
            node_.task = NULL; // 标志位：表示任务生命周期结束了
        }    
    }

    mutex_.lock();   // 这里是为了等待dispatch流程保证exchange已经结束，
    mutex_.unlock(); // 否则资源被释放就不能访问成员变量了。也是非常常用的技巧！
    this->__WFTimerTask::handle(state, error); // 里边会释放资源，并发起任务流下一个任务
}
```

第二类情况，如果用户调用**cancel**先发生呢？

1. 刚才提到，poller层面的`poller_del_timer()`会先置上`removed`，并从定时器数据结构中删掉，这样就不会被**stop**流程再去处理了；
2. 当然如果同一时间发生了**stop**，当前**cancel**流程已经把节点从poller的数据结构里删了，则还需要负责去调用一下Task的回调；
3. 在Task层面，需要由**cancel**流程负责调用用户的callback()，同时回收timer的资源；

// 可以加一个图。。。

```cpp
void __NamedTimerMap::cancel(const std::string& name, size_t max)               
{
    ...
    // 锁内，拿出命名为name的timer队列
    timers = __get_object_list<TimerList>(name, &root_, false);                 
    if (timers)
    {
        do
        {
            if (max == 0) // 从map中删除最多max个
                return;

            // 从该名字对应的队列中删除该timer
            node = list_entry(timers->head.next, struct __timer_node, list);    
            list_del(&node->list);

            // 标识位：exchange。如果是第二次exchange，会调到task自身的cancel()从poller中删掉它
            if (node->task->flag_.exchange(true))
                node->task->cancel();

            // 标志位：表示生命周期正确结束，资源已经被回收，否则timeout流程或析构函数需要做回收
            node->task = NULL;                                                  
            max--;                                                              
        } while (!timers->empty());                                             

        rb_erase(&timers->rb, &root_);
        delete timers;
    }
}
```

### 6. 异步任务的发起时机是个谜

还是回到这张图：

// wokrflow_timer_3.png

我们假设的是任务先创建好，再被发起。那如果任务任务还没有被发起，甚至我们不想发起呢？

1、任务可以在被发起前取消

实际上我们把一个timer放到一个任务流图中，我们并不能确定它被发起的准确的时机，但我们依然允许先**cancel**它。

// wokrflow_timer_6.png

这时候我们就需要上述的标志位`exchange`来做互斥了。`exchange`是个`std::atomic<tool>`，初始化为**false**，用户已经手动**cancel**过之后，任务可能在任务流中才会被发起**dispatch**。因此必须保证**dispatch**过才能释放这个timer的资源：

```cpp
/*** 由任务流发起、或者用户手动start起来 ***/
void __WFNamedTimerTask::dispatch()
{    
    int ret;    
    
    mutex_.lock();    
    ret = this->scheduler->sleep(this); // 先把定时任务交给poller
    if (ret >= 0 && flag_.exchange(true)) // exchange一下。如果是第二个调用exchange的人，会拿到true
        this->cancel(); // 说明发起之前已经有人cancel过了，立刻从poller中删除即可
    
    mutex_.unlock();    
    if (ret < 0)    
        this->handle(SS_STATE_ERROR, errno);
}
```

这里有两个要注意的点：
- 通过`sleep()`交给poller之后，用户的callback可以在网络线程中处理，而不是当前线程立刻处理，这样可以递归爆栈问题；
- 如果过期时间非常短，`sleep(`)之后是随时有可能到期并回到`__WFNamedTimerTask::handle()`的！因此前面对**mutex_.lock()**和**mutex_.unlock()**等的就是这里的`flag_.exchange(true)`执行结束；

2. 任务甚至可以不发起

而如果创建完之后不想发起，Workflow统一的接口是需要调用一下`task->dismiss()`，以释放task资源，调用析构函数等。

```cpp
/*** 命名定时任务的析构函数，异步任务需要注意处理各种情况的资源回收 ***/
virtual ~__WFNamedTimerTask()
{
    if (node_.task) // 标志位：如果没有置空，说明任务没有发起过。需要从全局map中删掉这个timer
    {
         std::lock_guard<std::mutex> lock(__timer_map.mutex_);
         if (node_.task) // 锁内再检查一次
             timers_->del(&node_, &__timer_map.root_);
    }
}
```

7. 总结

最后贴一段代码看看，一个高并发1s定时返回的server代码，用Workflow实现可以多简单：

```cpp
int main()
{
    WFHttpServer server([](WFHttpTask * task)
    {
        task->get_resp()->append_output_body("<html>will response after 1s</html>");
        auto timer = WFTaskFactory::create_timer_task(1, 0, nullptr);
        series_of(task)->push_back(timer);
    }); 

    if (server.start(1412) == 0)
    {   
        getchar();
        server.stop();
    }
    return 0;
}
```

本篇介绍了如何优雅地处理异步任务的：**创建、发起、回调、取消、进程退出**多种并发行为，其中包含了许多常用的技巧！不记得的小伙伴自行翻回去再看一遍._.

当然这在并发的世界中还只是冰山一角，因为很有可能写下某一句话的时机不对，任务结束了程序就GG了。

欢迎小伙伴提出宝贵的建议\^0^/ 

最后！

先前咨询过如何参与到Workflow开发中的小伙伴，你们的任务来了！！！

// wokrflow_timer_7.png

**可取消的定时器功能，目前还不支持Windows版本，欢迎熟悉iocp的小伙伴参与开发～**

不好意思上来就是个大boss。以及下次一定为新手村带来更多任务协助同学们打怪升级～感恩！
