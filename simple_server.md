# 10行C++代码实现高性能Http服务器

是不是觉得C++写个服务太累，但又沉迷于C++的真香性能而无法自拔？今天我来带你一起挑战，10行C++代码实现一个高性能的Http服务器，轻轻松松QPS几十万。Linus说：talk is cheap，show me the code ↓

```cpp
int main() {
    WFHttpServer server([](WFHttpTask *task) {
        task->get_resp()->append_output_body("<html>Hello World!</html>");
    });
    if (server.start(8888) == 0) {
        getchar(); // press "Enter" to end.
        server.stop();
    }
    return 0;
}
```

这个server使用了**Workflow**，安装编译都非常简单，以Linux为例，把代码拉下来，一行命令即搞定编译：

```sh
git clone https://github.com/sogou/workflow
cd workflow
make && make tutorial
```
tutorial文件夹里的helloworld示例就可以直接run起来，浏览器即可访问。实现短短10行代码，一次满足你三个愿望：

- 帮你收发和解析Http协议
- 收到消息之后回复一句Hello World!
- 在8888端口run起来

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
4. 逻辑在一个函数中（即上面的lambda），表示收到消息之后要做的事情，这里填了一句“Hello World!”;
5. Server启动和退出使用``start()``和``stop()``两个简单的api，而中间要用``getchar();``卡住，是因为**Workflow**是个**纯异步**的框架。

而**纯异步**，就是这个Http服务器的高性能所在：

想象一样，如果我们收到请求之后在这个函数里做了一些阻塞的事情（比如等锁、sleep或者忙碌的计算等），那么再有用户请求我的时候，我就没有线程去处理新用户了。

所以，**第一，我们希望server是多线程提供服务的。**

但再多的线程也可能会被霸占完的时候。我们需要无论server函数想要做任何耗时的操作，都不会影响到网络线程。

所以，**第二，我们希望网络线程和执行线程有优秀的调度策略。**

如果服务只打算支持一万的QPS，其实底层怎么实现都很简单，但如果我们希望十万，甚至接近百万，则我们对server底层做收发的I/O模型有非常高的要求。

所以，**第三，以linux为例，我们希望对epoll的封装高效好用。**

带着以上高吞吐的需求，我们来看看**Workflow**内部的实现是怎样的：

<img src="https://raw.githubusercontent.com/wiki/holmes1412/holmes1412/workflow_network.png"  width = "719" height = "520" alt="workflow_network_hierachy" align=center />

基于以上的架构，基于**Workflow**的server轻轻松松就可以达到几十万QPS，高吞吐、低成本、开发快，简直是居家旅行开发代码必备的良药！我们以数据说话，并且请来了名誉全球的高性能Http服务器nginx和国内开源框架先驱brpc一起做参考，看一下固定数据长度下QPS与并发度的关系：

<img src="https://raw.githubusercontent.com/wiki/sogou/workflow/img/benchmark-01.png"  width = "719" height = "520" alt="workflow_network_hierachy" align=center />

以上是在同一台机器上用相同的变量做的wrk压测，具体可以到github查看机器配置、参数及压测工具代码。当数据长度保持不变，QPS随着并发度提高而增大，后趋于平稳。此过程中**Workflow**一直有明显优势， 高于**nginx**和**brpc**。 特别是数据长度为64和512的两条曲线， 并发度足够的时候，可以保持50W的QPS。

**Workflow**能在开源大半年在github上收获4k星星的认可，当然是除了**简单**和**高性能**以外，还有其他许多的特点，如果你对其他使用场景还有所好奇，或者希望尝试压测一下感受高QPS带来的心跳加速，那么欢迎点击[Worklfow](https://github.com/sogou/workflow)的github猎奇更多脑洞大开的功能和用法。
