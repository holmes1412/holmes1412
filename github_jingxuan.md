# 新力作推荐：Workflow

**Workflow**是新一代基于**任务流模型**的C++**异步调度编程范式**，性能优异、生态丰富、通讯计算融为一体，解决了C++下异步开发的多个难点，目前已经是搜狗公司级C++标准，应用于搜狗大多数C++后端服务。

主要特点：

- 简单易上手，无依赖（适合初学者）
- 性能和稳定性优异
- 丰富的通用协议实现：HTTP、Redis、MySQL、Kafka、WebSocket等
- 统一计算、网络、文件IO等异步资源
- 任务流管理（串行、并行、DAG）
- 一致的解决方案形成一套完备的编程范式

我们以实际代码说话，如何用C++写一个简单的**HttpServer**：

```cpp
#include <stdio.h>
#include "workflow/WFHttpServer.h"

int main()
{
    WFHttpServer server([](WFHttpTask *task) {
        task->get_resp()->append_output_body("<html>Hello World!</html>");
    });

    if (server.start(8888) == 0) { // start server on port 8888
        getchar(); // press "Enter" to end.
        server.stop();
    }

    return 0;
}
```

在**Workflow**的世界中：**程序=协议+算法+任务流**，其中最大的创新点就在于任务流，不同异步资源都可以被组装起来：

// picture

高吞吐、低成本、开发快，并且轻松达到**几十万QPS**！以下与**nginx**和**brpc**在固定数据长度下的QPS与并发度对比中，**Workflow**都有明显的优势：

<img src="https://raw.githubusercontent.com/wiki/sogou/workflow/img/benchmark-01.png" alt="workflow_network_hierachy" align=center />

开源项目地址：https://github.com/sogou/workflow

开源项目组织：搜狗架构团队

//loonggg.android@foxmail.com
