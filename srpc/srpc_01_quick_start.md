# 《快速入门SRPC》

> 开源2年半了，一直都还没给SRPC系统地写过什么文章。趁着最近发布了srpc构建小工具，也给SRPC写几篇交流学习文档，希望单独的每一篇都能让不同程度的小伙伴有所收获~~~

## 1. 从srpc小工具开始

最近给**SRPC框架**做了一个**小工具**：**用于快速构建Workflow和SRPC项目的脚手架**，旨在**降低项目使用门槛**，解决大部分零基础开发者第一次面对**cmake文件编写**、lib的**依赖**、**编译与运行环境**等容易遇到的问题。

**srpc小工具**，让开发者的三个步骤：**构建 - 编译 - 运行**，都变得更简单（叉腰！

**SRPC地址：[https://github.com/sogou/srpc](https://github.com/sogou/srpc)**

另外，懒了好久没po文，新小伙伴可能比较多，以下补充一些可以跳过的背景知识。

**SRPC**是一个**轻量级、企业级、性能优异**的RPC框架，代码结构精巧解耦合，跟随源码看请求过程一气呵成，非常适合用来**学习**RPC架构，部署**使用**也都比较方便。

而**SRPC**又是基于**Workflow**开发的，**Workflow**目前已经是一个万星项目、成为**Debian / Ubuntu Linux / Fedora**等系统的自带安装包，也获得了大家还不错的口碑，所以就不多介绍了。等这列文章完结之后，我会再开展**Workflow**的学习系列，如果对底层网络模型、计算调度做法、任务流设计等感兴趣的小伙伴，需要再给一丢丢耐心等待～

**Workflow地址：[https://github.com/sogou/workflow](https://github.com/sogou/workflow)**

## 2. 一行命令构建起你的项目

### 2.1 源码位置

我们把上述的项目clone下来，并打开tools目录，就可以编译出我们的srpc小工具。工具名字也叫**srpc**，但是是小写的~

```
git clone --recursive https://github.com/sogou/srpc.git
cd srpc/tools && make
```

这个小工具和SRPC框架目前还没有关系，所以即使本地没有安装SRPC所需要的**protobuf**或者没有--recursive拉**submodule**下来，也依然可以编译。唯一需要的是**cmake** 3.6及以上的版本。

### 2.2 运行工具

我们先把这个srpc小工具**运行起来**，可以看到它第二个参数**COMMAND：表示支持什么命令**。

```
./srpc
```
```
Description:
    Simple generator for building Workflow and SRPC projects.

Usage:
    ./srpc <COMMAND> <PROJECT_NAME> [FLAGS]

Available Commands:
    http        - create project with both client and server
    redis       - create project with both client and server
    rpc          - create project with both client and server
    proxy      - create proxy for some client and server protocol
    file           - create project with asynchronous file service
    compute - create project with asynchronous computing service
```

这些COMMAND包括了我们最常用的场景，适合入门了解服务器编程最简单的内容。

### 2.3 一行命令构建项目

第三个参数是项目名，**我们先用一行简单的命令**，构建出一个Http服务器与客户端。

```
./srpc http my_project
```

```sh
Success:
      make project path " my_project/ " done.

Commands:
      cd my_project/
      make -j

Execute:
      ./server
      ./client
```

<img src="https://raw.githubusercontent.com/wiki/holmes1412/holmes1412/srpc/srpc_01_tools_01_build.gif">

可以看到提示**Success!**

###  2.4 第一个项目的编译和运行

我们按照上述的提示看到：**my_project**目录已经在本地目录下创建，并给出了：

- 编译的命令：make
- 运行的命令：分别在两个终端上执行./server 和 ./client

```
cd my_project
make
```
执行`ls -all` 一下可以看到，两个可执行文件已经编译出来了。我们分别在两个终端运行`./server`和`./client`：

```
./server 
```
```
Http server started, port 8080
http server get request_uri: /client_request
peer address: 127.0.0.1:65313, seq: 0.
```

client运行起来后会给server发一个请求，然后server会打印出上面显示的最后两行，然后client收到回复之后也会打印下面的两行：

```
./client 
```
```
Http client state = 0 error = 0
<html>Hello from server!</html>
```

<img src="https://raw.githubusercontent.com/wiki/holmes1412/holmes1412/srpc/srpc_01_tools_02_execute.gif">

### 2.5 模块依赖，是C++项目的第一道门槛

即使刚才git clone项目时没有加--recursive拉取依赖的submodule，或者srpc的lib没有编译（按上述的步骤的话，是还没有的～)，那么**工具会自动做一些初始化的工作**。

当然，目前C++跟GO等其他语言比起来在构建方面还是薄弱了一点。如果大家还没有安装protobuf，或者系统的版本太旧、导致编译SRPC时所依赖的protobuf版本与链接时不一样，那么可以先**使用源码编译protobuf**。

这里找了一个不太新也不太旧的版本：

```sh
git clone -b 3.20.x https://github.com/protocolbuffers/protobuf.git protobuf.3.20
cd protobuf.3.20
sh autogen.sh
./configure
make -j4
make install
```

然后我们就可以愉快再试一下上述步骤了～

## 3. 一个脚手架项目包含了什么？

我们执行`tree`命令，查看这个项目里的文件结构。

需要我们关注的有这些：

<img src="https://raw.githubusercontent.com/wiki/holmes1412/holmes1412/srpc/srpc_01_tools_03_tree.jpg">

### 3.1 编译文件

脚手架小工具目前还是使用cmake进行编译，后续计划支持bazel和xmake。

`GNUmakefile`包了一层cmake命令，让我们可以执行`make`就编译出项目，这个文件我们不需要关心。

打开`CMakeLists.txt`，可以看到一共32行，包括了：
1. 寻找依赖路径的写法
2. include和link的写法
3. 编出执行文件

开发者可以根据里边的注释自行修改，**即使不常用C++的开发者也可以边试边学**。

### 3.2 client

client会读取`client.conf`作为它的配置文件，主要是指定要访问的目标是什么。

我们打开`client_main.cc`，可以看到脚手架默认生成的client只有60多行，一共做了3件事：

```cpp
int main()
{
    // 1. 初始化，这个实现也在源码中，主要是调用config.load("./client.conf")
    init();

    std::string url = std::string("http://") + config.client_host() +
                      std::string(":") + std::to_string(config.client_port());
 
    // 2. 构造一个http task并且填回调函数
    WFHttpTask *task = WFTaskFactory::create_http_task(url,                        
                                                        config.redirect_max(),  
                                                        config.retry_max(),        
                                                        callback);
    // 3. 把task运行起来
    task->start();                                                                 

    wait_group.wait();
    return 0;
}
```

可以看到，这个和workflow的tutorial中的例子是一样的，需要填的**callback**函数也在文件中。

### 3.4 server

我们打开`server_main.cc`，50多行的代码，也是做了3件事，可以看到**和上面的client是非常对称的**：

```cpp
int main()
{
    // 1. 初始化，这个实现也在源码中，主要是调用config.load(“./server.conf")
    init(); 

    // 2. 造一个server，填好处理函数
    WFHttpServer server(process);                                                  

    // 3. 把server运行起来
    if (server.start(config.server_port()) == 0)                                   
    {
        fprintf(stderr, "Http server started, port %u\n", config.server_port());
        wait_group.wait();
        server.stop();
    }
    else
        perror("server start");

    return 0;
}
```

process函数也在源码中，开发者可以尝试修改，进行不同的行为处理。示例中的行为就是回复一个 " Hello from server! "

### 3.5 配置文件

**配置解析**并不是Workflow和SRPC项目自带的，但是脚手架项目增加了这个功能。

我们目前使用的配置文件都是**json格式**，和配置解析相关的都放到了config目录中。除了**client.conf**和**server.conf**以外，我们还多加了一份**full.conf**，用来指引Workflow和SRPC目前支持的配置项，开发者可以通配置文件，快速了解我们还可以用什么功能。

比如框架的全局配置：

```json
{                                                                                  
  "server":                                                                        
  {
    "port": 8080                                                                   
  },
                                                                                   
  "client":                                                                        
  {
    "remote_host": "127.0.0.1",
    "remote_port": 8080,
    "retry_max": 1,
  },

  "global":
  {
    "poller_threads": 4,
    "handler_threads": 20
  }
}
```

熟悉的开发者可能接触过**Workflow**的**upstream**，以及**trace**和**metrics**等监控数据的**上报插件**，这些都可以在配置文件中指定并**一键加载**，帮开发者接管外部生态，真正实现脚手架的能力。

## 4. 命令大全

经过以上介绍，应该可以基本掌握怎么快速构建和运行一个自己的小项目。接下来我们进一步解锁这个srpc小工具，每个**COMMAND**都是一个**二级命令**。

### 4.1 rpc命令

构建一个以protobuf或者thrift作为IDL的多协议RPC项目。

如果先前没有了解过SRPC的小伙伴，有空可以围观一下这份wiki：[SRPC架构介绍 - Sogou基于Workflow的自研RPC框架](https://github.com/sogou/srpc/blob/master/docs/wiki.md)，也可以这里用一句话简述一下：

SRPC框架支持：
- 多种**协议**
- 多种**IDL**
- 多种**数据格式**
- 多种**压缩算法**

我们的client和server只需要保证**以同样的协议进行通信**，而其余的东西交给SPRC框架帮你处理就好，最终开发者接触到的就是我们的**IDL所约定的接口**，比如在`xxx.proto`文件中长这样：

```proto
service rpc_test {                                                                 
    rpc Echo(EchoRequest) returns (EchoResponse);                                  
};
```

其中，支持的RPC协议包括：**SRPC、SRPCHttp、BRPC、TRPC、TRPCHttp、Thrift、ThriftHttp**。

这些东西都可以在构建时通过参数指定。

我们执行`./srpc rpc`，就可以看到rpc命令支持的参数：

<img src="https://raw.githubusercontent.com/wiki/holmes1412/holmes1412/srpc/srpc_01_tools_04_rpc.jpg">

我们尝试以默认方式构建一个RPC项目。也可以使用`-f`指定IDL文件进行构建，这会使用`srpc_generator`去进行**代码生成**。

```
./srpc rpc rpc_project
```

打开之后，可以看到和Http相比，有如下区别：
1. 多了一个`rpc_project.proto`
2. server_main.cc和client_main.cc分别变成了S**RPCServer**和**SRPCClient**；
3. CMakeLists.txt也变复杂了，因为需要依赖protobuf和snappy等压缩库；

但我们依然可以通过`make`把项目编译出来。运行方式与前面类似，不再赘述。

### 4.2 redis命令

这个命令主要用于构建**redis协议**的server和client，由于Workflow的协议对server和client来说都是对等的，因此基于Workflow实现的redis server依然非常简洁高效。

这个例子中，client发出的请求是`set k1 v1`，server收到任何内容都回复一个`OK`。并且client.conf中增加了用户名和密码的项，开发者可以通过修改配置，用这个client访问其他任意的redis server。

### 4.3 proxy : 代理服务器

代理服务器顾名思义，就是可以多构建一个proxy，我们可以用client去访问proxy，并由proxy去转发给server，这中间proxy就可以做很多事情，包括：更改协议、内容校验等等。

一个常见的场景是，我们的现有业务是客户端**发出TRPC协议**，而**需要访问SRPC协议的服务器**时，则可以构建出一个TRPC-SRPC的proxy，并且让大家使用统一的proto文件约定好请求，则proxy就可以直接做转发。

<img src="https://raw.githubusercontent.com/wiki/holmes1412/holmes1412/srpc/srpc_01_tools_05_proxy.jpg">

我们执行如下命令，用`-c`指定client端的协议，用`-s`指定server端的协议：

```
./srpc proxy proxy_project -c trpc -s srpc
```

然后还是按照前面所述的编译和运行。这里我们分别运行起来`./server`，`./proxy` 和`./client`：
<img src="https://raw.githubusercontent.com/wiki/holmes1412/holmes1412/srpc/srpc_01_tools_06_proxy.gif">

而`proxy_main.cc`的实现，是**起了一个TRPCServer，并使用SRPCClient去转发请求**。感兴趣的小伙伴可以围观一下，其实SRPC项目的tutorial里也已经有这样的例子了：[tutorial-15-srpc_pb_proxy.cc](https://github.com/sogou/srpc/blob/master/tutorial/tutorial-15-srpc_pb_proxy.cc)

### 4.4 file : 文件服务器

文件服务器通过异步IO读取，也是我们常用的功能，这里不再赘述，对实现感兴趣的小伙伴欢迎查看原先的一篇文章：[《Workflow编程小示例4: 转发服务器与series上下文的使用》](https://zhuanlan.zhihu.com/p/390437518)

我们通过`./srpc file file_project`构建一下项目，我们可以通过curl命令去读取想要的文件，例如`curl localhost:8080/index.html` 就可以读取到指定root目录下的index.html文件。

<img src="https://raw.githubusercontent.com/wiki/holmes1412/holmes1412/srpc/srpc_01_tools_07_file.jpg">

可以看到，多了一个`html`的目录，里边放了`index.html`, `404.hmtl`和`50x.html`。如果使用过其他Http服务器的小伙伴应该不陌生，这是常见的用法。

```sh
.
├── CMakeLists.txt
├── GNUmakefile
├── config
│   ├── Json.cc
│   ├── Json.h
│   ├── config.cc
│   └── config.h
├── file_service.cc
├── file_service.h
├── html
│   ├── 404.html
│   ├── 50x.html
│   └── index.html
├── server.conf
└── server_main.cc

3 directories, 13 files
```

我们还可以通过配置文件去指定具体错误码对应的错误页面，而错误页面和其他文件一样，都是通过异步IO的方式读取，不会阻塞server当前的处理线程。

熟悉的error_page它来了：

```json
{     
  "server":     
  {     
    "port": 8080,     
    "root": "./html/",     
    "error_page" : [     
      {     
        "error" : [ 404 ],     
        "page" : "404.html"     
      },     
      {     
        "error" : [ 500, 502, 503, 504],     
        "page" : "50x.html"     
      }     
    ]     
  }     
}
```

### 4.5 compute : 计算服务器

计算服务器也是通过go_task发起计算任务，不会阻塞当前线程。实现原理欢迎参考：[《WF编程小示例6: 计算型服务器与计算任务》](https://gitee.com/sogou/workflow/issues/I40VAX)

## 5. 其他

srpc小工具的基本用法就介绍完了，但是由于它刚刚面世，之后会支持命令还会更多，支持的配置也会更多，比如error_page这种大家习惯的配置用法，22同学都会想想怎么加上～

希望这个小工具可以减少开发者第一次接触项目时，“**构建 - 编译 - 运行**”所面临的困难，而SRPC项目之后也将致力于**降低开发者使用门槛**，包括优化**依赖库和submodule**、**提供更多编译方式**、**支持更多好用的生态插件**等等。

当然这系列的学习文章也是**降低使用门槛**、**陪伴小伙伴们学习代码**、**获取大家反馈促进交流**的重要一环，之后争取周更，把这系列的坑填完！所以想要看什么内容或者对srpc小工具有什么建议要提的都得赶紧了~

最后附上github上的文档：[https://github.com/sogou/srpc/blob/master/tools/README.md](https://github.com/sogou/srpc/blob/master/tools/README.md)
