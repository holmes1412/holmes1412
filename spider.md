# 为什么spider抓取是程序员基本技能？

### 一、什么是spider

**Spider抓取**，可以说是我们需要从互联网世界获取数据的最基本方法。以大家最常用的``HTTP``协议为例，抓取本质上是一个**URL**到一个**网页**的转换过程。而有了基本的抓取功能，再搭配上我们想要的规则和策略，这就是一个上至内容检索下至数据分析都可以实现的spider。

继上一次介绍用**Workflow**可以10行C++代码写一个服务器，今天继续给大家用十几行C++代码来实现一个高性能spider客户端！！！

```cpp
//[spider.cc]
#include "stdio.h"
#include "unistd.h"
#include "workflow/HttpMessage.h"
#include "workflow/WFTaskFactory.h"

int main (int argc, char *argv[])
{
    const char *url = "https://github.com/sogou/workflow";
    WFHttpTask *task = WFTaskFactory::create_http_task (url, 2, 3,
            [](WFHttpTask * task) { 
                fprintf(stderr, "%s %s %s\r\n",
                        task->get_resp()->get_http_version(),
                        task->get_resp()->get_status_code(),
                        task->get_resp()->get_reason_phrase());
    });
    task->start();
    pause();
    return 0;
}
```
只要安装好了workflow，以上代码即可以通过以下命令编译出一个简单的spider：
```sh
g++ -o spider spider.cc --std=c++11 -lworkflow -lssl -lcrypto -lpthread
```
根据http协议，我们执行这个可执行程序``./spider``，就会得到以下内容：
```sh
HTTP/1.1 200 OK
```
同理，我们还可以通过其他api来获得抓取回来的其他http header和http body，一切内容都在这个``WFHttpTask``中。而因为Workflow是个异步调度框架，因此这个任务发出之后，不会阻塞当前线程，从根本上保证了我们的spider的高性能。

接下来给大家详细讲解一下接口和原理～

### 二、如何做抓取

#### 1. 创建抓取任务

上述demo可以看到，我们的抓取是通过Workflow的一个Http异步任务来实现的，创建任务的接口如下：
```cpp
WFHttpTask *create_http_task(const std::string& url,
                             int redirect_max, int retry_max,
                             http_callback_t callback);
```
第一个参数就是我们要抓取的URL。对应的，在一开始的示例中，我们的**redirect_max**是2次，而**retry_max**是3次。第四个参数是一个回调函数，示例中我们用了一个lambda，由于Workflow的任务都是异步的，因此我们处理抓取结果这件事情是被动通知我们的，就会调起这个回调函数，格式如下：
```cpp
using http_callback_t = std::function<void (WFHttpTask *)>;
```

#### 2. 填写header并发出

我们的网络交互无非是**请求-回复**，对应到spider上，在我们创建好了task之后，我们有一些时机是处理**请求**的，在``HTTP``协议里，就是在header里填好协议相关的事情，比如我们可以通过Connection来指定希望得到建立``HTTP``的**长连接**，以节省下次建立连接的耗时，那么我们可以把**Connection**设置为**Keep-Alive**。示例如下：
```cpp
protocol::HttpRequest *req = task->get_req();
req->add_header_pair("Accept", "*/*");
req->add_header_pair("User-Agent", "Wget/1.14 (gnu-linux)");
req->add_header_pair("Connection", "Keep-Alive");
task->start();
```
最后我们会把设置好**请求**的任务，通过``task->start();``发出。最开始的``spider.cc``示例中，有一个``pause();``语句，是因为我们的任务发出后是非阻塞的，当前线程不暂时停住就会退出，而我们希望等到回调函数回来，因此我们可以用多种暂停的方式。

#### 3. 处理抓取结果

文章开头的demo里，为了方便读者，回调函数是一个lambda。现在我们把这个回调函数独立拎出来看看：
```cpp
void spider_callback(WFHttpTask *task)
{
    protocol::HttpRequest *req = task->get_req();
    protocol::HttpResponse *resp = task->get_resp();
    std::string name;
    std::string value;

    // 获取http回复中的header
    protocol::HttpHeaderCursor req_cursor(req);
    while (req_cursor.next(name, value))
        fprintf(stderr, "%s: %s\r\n", name.c_str(), value.c_str());
    fprintf(stderr, "\r\n");

    // 获取http回复中的body
    const void *body;
    size_t body_len;
    resp->get_parsed_body(&body, &body_len); 
    fprintf(stderr, "%.*s", body_len, (char *)body);
}
```
一个抓取结果，根据``HTTP``协议，会包含三部分：
- 消息行(包括协议、状态码、状态码描述)
- 消息头heaer
- 消息正文body

其中第一部分在``spider.cc``这个demo里已经展示，这里给大家看看如何获取header和body。

header需要通过一个``HttpHeaderCursor``遍历获得，而body可以通过``get_parsed_body(const void **body, size_t *size)``拿到，这是一个零拷贝的接口，如果我们抓取后希望结果存下来，需要在这个回调函数退出前把内容拷走。

### 三、解锁更多功能

有了一个简单的抓取任务，我们还会有哪方面的需求呢，这里可以结合实际情况与大家分享：
1. **并行**地抓取多个URL，并且在全部抓取回来后做一些分析；
2. **按顺序**或者按指定速度抓取某个站点的内容，避免抓取过快被封禁；
3. 抓取必须是**异步**的，以避免网络延迟高的情况下占用当前线程的问题；
4. spider遇到**redirect**可以自动帮我做跳转，一步到位抓取到最终结果；
5. 我们可以实现**代理服务器**帮他人抓取，而他人抓取的网页可以是``HTTPS``的;

以上这些需求，除了需要框架本身具有高性能以外，还要求框架对于抓取任务的编排有超高的灵活性，以及对实际需求（比如redirect、ssl代理等功能）有非常接地气的支持，这些``Workflow``都已经实现。如此快速就能实现一个功能丰富的高性能spider，赶紧点击链接[https://github.com/sogou/workflow](https://github.com/sogou/workflow)尝试一下吧！
