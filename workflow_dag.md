# 用C++ Workflow构建DAG，轻松实现高性能异步调度

异步代码往往因为处理并发、条件、错误、超时而加倍复杂，有向无环图（DAG）是解决代码逻辑依赖的极佳模型。

Workflow作为一个异步编程框架同样也有graph_task，其中Graph和Node的都是基于现有任务流模式而实现，非常巧妙有趣，浑然天成。它让任务满足我们构建的DAG逻辑运行的同时，又可以让有任务按照底层资源约束最大化地被并发调度起来，充分发挥Workflow的精髓。

https://github.com/sogou/workflow

## 1. DAG解决什么问题

以线上服务经典的大扇出场景为例：我们想要发出很多个网络请求，有一部分失败了没关系，只要大部分都成功了就行，但需要控制整体耗时。

这里其实包含了两个条件，满足任何一个都可以让这个任务流往下走：

- 大部分下游任务结束
- 计时器结束

熟悉Workflow的开发者可以思考一下，只用串行SeriesWork/并行ParallelWork是否足以解决上面的场景呢？

答案是不够的，因为这里有个“或”的条件。

但有向无环图可以。（当然用WFCounterTask也是可以的）

如果说串行可以表达为是1:1，并行是N:1（所有依赖结束后逻辑收敛到一个代码块），那么DAG就是N:M，是更加广义的结构，星型~~、。可以说graph_task是Workflow的任务流最自由的表达。~~

更重要的是，DAG除了任务，还有图。在Workflow中，图本身也是一种任务，这样可以让这个图被放到任何一个任务流中，实现代码逻辑的隐藏和封装。

~~才能做到进程代数（Process Algebra）中~~

## 2. 这样组建DAG

Workflow的DAG提供了一些有趣的语法糖，在2022年的C++ Summit上我们也特地po过，以下和大家一起看看~

// 图

### 2.1 创建图任务

```cpp
{
    WFGraphTask *graph = WFTaskFactory::create_graph_task([](WFGraphTask *) {
        printf("Graph task complete. Wakeup main process\\n");
        wait_group.done();
    });
}
```

可以看到，图任务的类型为`WFGraphTask`，创建函数只有一个参数，即任务的回调。显然一个新建的图任务，是一张空图。

### 2.2 创建图节点

我们需要通过之前创建的4个普通任务（http_task, timer, go_task, write_task），产生4个图节点：

```cpp
{
   /* Create graph nodes */
    WFGraphNode& a = graph->create_graph_node(http_task);
    WFGraphNode& b = graph->create_graph_node(timer);
    WFGraphNode& c = graph->create_graph_node(go_task);
    WFGraphNode& d = graph->create_graph_node(write_task);
}
```

`WFGraphTask`的create_graph_node()接口，产生一个图节点`WFGraphNode`并返回节点的引用，用户通过这个节点引用来建立节点之间的依赖。

如果我们不为节点建立依赖直接运行图任务，那么显然所有节点都是孤立节点，将全部并发执行。

### 2.3 建立依赖

可以这样：'-->'
```cpp
{
   /* Build the graph */
    a-->b;
    a-->c;
    b-->d;
    c-->d;
}
```

也可以这样：’<—-‘
```cpp
{
    a-->b-->d;
    a-->c-->d;
}
```

等价于这样：
```cpp
{
    d<--b<--a;
    d<--c<--a;
}
```

当然还可以这样：
```cpp
{
    d<--b<--a-->c-->d;
}
```

接下来直接运行graph，或者把graph放入任务流中就可以运行啦，和一般的任务没有区别。

当然，把一个图任务变成另一个图的节点，也是完全正确的行为。

## 3. 实现原理

对图和节点，可以梳理出以下功能：
- 节点任务(WFGraphNode)：在前序节点都完成时，调起它接管的task；在task结束时后，告知后序节点；
- 图任务（WFGraphTask）：整个图被调起时，并发调起所有节点，让他们按依赖运行；整个图结束之后，调用用户回调函数；

那么很显然，可以分别用WFCounterTask和ParallelWork来实现，我们已经快要和作者一起设计出DAG的模块了！

- WFGraphNode派生自WFCounterTask，数够N个数，当前任务流就往下走；
- WFGraphTask派生自WFGenericTask，内部发起一个ParallelWork，用于管理Node；

有些小伙伴可能对WFGenericTask比较陌生，但其实，它是Workflow多种基础任务中的万金油。据不完全统计（作者手动grep._.），WFGenericTask是最多子类的一个基础任务。

### 3.1 先看图：WFGraphTask

```cpp
class WFGraphTask : public WFGenericTask
{
public:
    WFGraphNode& create_graph_node(SubTask *task); // 最常用的接口
    ...

protected:
    virtual void dispatch(); // 这两个接口要重新实现
    virtual SubTask *done(); // 因为图会被调起两次~

protected:
    ParallelWork *parallel; // 最重要的内部数据结构
    std::function<void (WFGraphTask *)> callback; // 每个图任务都有回调
    ...
};
```

我们看一下把某个task交给这个图的时候会发生什么：
```cpp
WFGraphNode& WFGraphTask::create_graph_node(SubTask *task)
{
    // 1. 创建一个节点
    WFGraphNode *node = new WFGraphNode;
    // 2. 为节点单独创建一个series任务流，并且头尾都是它自己。这意味着它也是会被调起两次
    SeriesWork *series = Workflow::create_series_work(node, node, nullptr);
    // 3. 节点中间接管着真正的异步任务
    series->push_back(task);
    // 4. 添加到本图的parallel并行任务中
    this->parallel->add_series(series);
    return *node;
}
```

上述实现中，我们需要关注两点：

1. 每调用一次create_graph_node()，相当于创建了一个s[ node_i → task → node_i]的任务流；
2. 每个node_i都被添加到parallel中，也就是说此时p{ s[node_0] | s[node_1] | … | s[node_n] }是完全并行的。

接下来看看如果图任务被调起，会发生什么：

```cpp
void WFGraphTask::dispatch()
{
    SeriesWork *series = series_of(this);

    if (this->parallel) // 作为标记位，第一次被调起时走这个分支
    {
        series->push_front(this); // 2. 再发起一次graph自己
        series->push_front(this->parallel); // 1. 先发起parallel
        this->parallel = NULL;
    }   
    else
        this->state = WFT_STATE_SUCCESS;

    this->subtask_done();
}
```

这样实现，是为了让我们的parallel能够被调度起来、同时保证callback调用的参数是WFGraphTask本身，所以parallel结束了之后WFGraphTask会再被拉起，用来回调。对应的，done()也要处理两次调用到的逻辑，感兴趣的小伙伴自行查阅~

### 3.2 再看节点：WFGraphNode

```cpp
class WFGraphNode : protected WFCounterTask
{
public:
    void precede(WFGraphNode& node) // 标记我是node的前序节点
    {   
        node.value++; // node的value表示了它有多少个前序节点
        this->successors.push_back(&node);
    }
    
    void succeed(WFGraphNode& node) // 标记我是node的后序节点
    {   
        node.precede(*this);
    }

protected:
    virtual SubTask *done(); // 这里需要重新实现

protected:
    std::vector<WFGraphNode *> successors; // 每人保存后面有谁

protected:
    WFGraphNode() : WFCounterTask(0, nullptr) { }
    virtual ~WFGraphNode();
    friend class WFGraphTask;
};
```

从上面就可以看到我们如何保存节点的边：
- 每个node都保存后面有具体**哪些**节点（successors）；
- 每个node都保存前面有**多少个**节点（而不关心具体是谁）；
- 一开始父类的count是0，表示它被调起之后就立刻可以运行下去；

那么如何调度起来呢？答案在`done()`的实现上：

```cpp
SubTask *WFGraphNode::done()
{
    SeriesWork *series = series_of(this);

    if (!this->user_data)
    {   
        this->value = 1;
        this->user_data = (void *)1;
    }   
    else
        delete this;

    return series->pop();
}
```
一开始，user_data肯定是空的，这意味着它的value会被置为1；
然后它接管的task就会被执行；
然后再到它第二次被dispatch()和done()，这时候它就通过析构函数，通知所有的后序节点：

```cpp
WFGraphNode::~WFGraphNode()
{
    if (this->user_data)
    {
        for (WFGraphNode *node : this->successors)
            node->WFCounterTask::count();
    }
}
```

4、最后

大名鼎鼎的Dijkstra、Ole-Johan Dahl和 C.A.R Hoare在《Structured Programming》一书中提到过，应该使用抽象的控制结构来组织代码。该书针对结构化并发其实还提出了许多约束，其突破性的思想在于，正是需要这些约束，而非功能的增加，才能让并发的程序得到清晰、明确的运行模式。

结构化并发的概念起源于70年代，比C++的诞生还要早很多。很遗憾的是C++一开始的设计中并没有在语言层面支持并发模型，以至于后续修修补补，但依然还没有打通语法和资源。Workflow一直用任务流贯穿着框架的所有调用，提出任务流作为结构化并发的载体，无论是

要实现优秀的并行模型往往不是加功能，相反，？

当然，基于DAG上还有很多事情可以做，比如原生的数据传递、组织DAG的低代码开发等，都是可以发挥想象力的空间！



