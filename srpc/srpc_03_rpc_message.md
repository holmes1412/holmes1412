# SRPC系列3: rpc_message源码解析
> 最近给SRPC项目写几篇学习文章，希望协助小伙伴通过这个轻量级的框架快速了解RPC相关内容。  
> 第三篇简述RPC协议、rpc_message模块源码，以及内部如何借助多派生等方法彻底解耦，实现高度代码复用。  
> **整体阅读预计5分钟。**  

本篇源码解析重点在多派生的结构讲解上，它可以让我们拥有多个父类的能力的同时，通过指定哪个接口调哪个实现，来充分复用父类的代码，因此非常适合于派生类型比较多、大家相互独立又平等的架构上使用。  

在此之前会补充RPC协议的基本描述，有基础的小伙伴可以直接跳到第二部分开始阅读。  

## 1. RPC协议的定位

我们日常开发所接触到的都是**应用层协议**，所以RPC协议的定位就是用来方便应用层的：
- **框架**，收发要简单且快捷，常用**header**去区分；
- **用户**，容易使用，**body**可以直接转成数据结构最好；

我们常用的**HTTP**是明文header，于是为了解析效率与传输考虑，**各RPC框架**会制定出自己的**二进制RPC协议**，和HTTP相比大约能快20%左右（当然大部分情况下咱们服务瓶颈还不到这儿~）

目前SRPC框架支持的协议有：**SRPC协议**、**BRPC协议**、**TRPC协议**、**Thrift协议**。比较热门的**gRPC协议**没有实现，是因为网络底层请求模式不太一样。

<img src="https://raw.githubusercontent.com/wiki/holmes1412/holmes1412/srpc/srpc_03_message_01.png" alt="图1: SRPC协议定义" align=center />
图1: SRPC协议定义

<img src="https://raw.githubusercontent.com/wiki/holmes1412/holmes1412/srpc/srpc_03_message_02.png" alt="图2: Http协议定义" align=center />
图2: Http协议定义

无论是RPC还是HTTP，body常用便于**序列化/反序列化**的数据格式（比如**json / xml / protobuf / thrift** / ...）。

## 2. 实现原理

SRPC框架对于上述**多种协议**、**多种IDL**等，**都是用同一套代码**>_<～不仅如此，对于**SRPCStd**协议和**SRPCHttp**协议也都是**同一套代码**。其中有非常精巧的**层次拆分**，做到彻底**解耦**，横向高度**复用**、纵向互相**独立**。

<img src="https://raw.githubusercontent.com/wiki/holmes1412/holmes1412/srpc/srpc_03_message_03.png" alt="图3: src/message/ 目录中的文件" align=center />
图3: src/message/ 目录中的文件

### 2.1 rpc_message

打开`rpc_message.h`，可以看到最底层的类。

```cpp
class RPCMessage
{
// 1. 通过重载函数来实现同时支持多种IDL，IDL一般是body
//    注意这里不是纯虚函数，说明可以只选一组实现
public:
 //pb
 virtual int serialize(const ProtobufIDLMessage *idl_msg);
 virtual int deserialize(ProtobufIDLMessage *idl_msg);

public:
 //thrift
 virtual int serialize(const ThriftIDLMessage *idl_msg);
 virtual int deserialize(ThriftIDLMessage *idl_msg);
 ...
};
```

因此如何同时支持IDL为多protobuf/thrift的答案就有了：**重载函数**。

此外`rpc_message.h`文件还有两个工具人，主要用来做子类接口约束：

```cpp
class RPCRequest
{
// 2. 约束子类必须实现序列化/反序列化header的接口
public: 
 virtual bool serialize_meta() = 0;
 virtual bool deserialize_meta() = 0;
 ...
};

class RPCResponse
{
public: 
 virtual bool serialize_meta() = 0;
 virtual bool deserialize_meta() = 0;

 // 3. Response出了了也有serialize/deserialize接口以外，还有常见的状态码和错误码
 virtual int get_status_code() const = 0;
 virtual int get_error() const = 0;
 virtual const char *get_errmsg() const = 0;

 virtual void set_status_code(int code) = 0;
 virtual void set_error(int error) = 0;

};
```

### 2.2 srpc消息格式

我们可以看到文件夹下有多个派生类，选取`rpc_message_srpc.h`看看SRPC协议：

```cpp
class SRPCMessage : public RPCMessage
{
public:
 // 4. 实际收发消息的序列化反序列化接口在这里实现
 int encode(struct iovec vectors[], int max, size_t size_limit);
 int append(const void *buf, size_t *size, size_t size_limit);

 // 5. header怎么按图1中定义的SRPC协议收发，也在这里实现
 bool serialize_meta();
 bool deserialize_meta();

 // 6. 由于SRPC协议支持两种IDL，因此两个重载函数都要实现
 int serialize(const ProtobufIDLMessage *pb_msg) override;
 int deserialize(ProtobufIDLMessage *pb_msg) override;
 int serialize(const ThriftIDLMessage *thrift_msg) override;
 int deserialize(ThriftIDLMessage *thrift_msg) override;

 // 实际header和body存放的成员变量
protected:
 // "SRPC" + META_LEN + MESSAGE_LEN + RESERVED
 char header[SRPC_HEADER_SIZE];
 RPCBuffer *buf;
 ...
};
```

可以看到，整个RPC协议的实现在这个类中。

这也解答了：**如何实现不同的RPC协议，比如SRPC和TRPC？**

每一种协议都从`RPCMessage`派生；并在`encode()`里把内存按照比如图1中的要求，填到要发出的网络包中；在`append()`中实现把网络包也按比如图1的格式收下来，那么就是一份完整的SRPC消息。

### 2.3 ProtocolMessage，Workflow的基本消息格式

我们为什么要这样定义`encode()`/`append()`接口，是有原因的。因为我们要使用Workflow的通信器进行收发。

底层基础决定上层设计，Workflow的通信器约定消息都从`ProtocolMessage`派生，并实现`encode()`/`append()`接口。

```cpp
// Workflow中的代码
class ProtocolMessage : public CommMessageOut, public CommMessageIn
{
private:
 virtual int encode(struct iovec vectors[], int max);

 /* You have to implement one of the 'append' functions, and the first one
 * with arguement 'size_t *size' is recommmended. */
 virtual int append(const void *buf, size_t *size);
 virtual int append(const void *buf, size_t size);
 ...
};
```

### 2.4 纯虚接口

回到SRPC框架，我们先从`SRPCMessage`再派生一层出来，分别用来表达**Request**和**Response**，主要为了根据SRPC协议本身的约定，来实现第一部分的RPCRequest和RPCResponse中的纯虚接口。

```cp
// 7. 限定了Request要用的接口。Request负责指定RPC中的service和method名
class SRPCRequest : public SRPCMessage
{
public:
 const std::string& get_service_name() const;
 const std::string& get_method_name() const;

 void set_service_name(const std::string& service_name);
 void set_method_name(const std::string& method_name);
};

// 8. 限定了Response要用的接口。Response负责表达状态码/错误码
class SRPCResponse : public SRPCMessage
{
public:
 int get_status_code() const;
 int get_error() const;
 const char *get_errmsg() const;

 void set_status_code(int code);
 void set_error(int error);
};
```

### 2.5 多派生

重点在于刚才说到Workflow的`encode()`/`append()`接口。我们刚刚明明是在`SRPCMessage`里实现，与**Workflow**的`ProtocolMessage`有什么关系呢？

为了联动起来，我们做了一个**多派生**。

我们以`SRPCStdRequest`为例，其中，**SRPC**表示SRPC协议；**Std**表示二进制；**Request**表示作为请求：

```cpp
// 从ProtocolMessage、RPCRequest、SRPCRequest多派生，则拥有了它们的所有能力
class SRPCStdRequest : public protocol::ProtocolMessage, public RPCRequest, public SRPCRequest
{
public: 
 // 9. Workflow通信器调用到ProtocolMessage::encode()时，就会走到这里，
 //  进而调用到刚才SRPCMessage::encode()填SRPC协议然后发出
 int encode(struct iovec vectors[], int max) override
 {
 return this->SRPCRequest::encode(vectors, max, this->size_limit);
 }

 // 10. 同理，fd上收到消息时，不停调用ProtocolMessage所约定好的append()
 //  然后按照SRPCMessage::append()的方式解析二进制
 int append(const void *buf, size_t *size) override
 {
 return this->SRPCRequest::append(buf, size, this->size_limit);
 }
 ...
};
```

其他接口也是类似的，简单一句话就是：**实现一个父类接口，内部指定调另一个父类方法。**

<img src="https://raw.githubusercontent.com/wiki/holmes1412/holmes1412/srpc/srpc_03_message_04.png" alt="图4: SRPC中的Message多派生结构" align=center />
图4: SRPC中的Message多派生结构

因此在这个场景下，多派生重要意义有两个：
1. 可以把**想要复用的**、且明确描述**独立功能**的代码放到父类中，不同的子类通过**组装**，从而实现不同的功能；
2. 父类的接口定义受底层框架约束，我们派生即可**拥有父类的能力**，同时也**接受框架调用父类的约束。**

### 2.6 打通Http：进一步代码复用

当然了如果我们还不需要子类多种实现的时候，总是很难看清为什么底层要这样拆。因此我们看看SRPC是怎么**即实现Std二进制又实现Http的**。

小伙伴们知道，母项目Workflow中有**HttpMessage的完整解析**，我现在不想抄过来，怎么办？

也是**多派生**解决，把上面的`ProtocolMessage`换成`HttpRequest`或`HttpResponse`即可：

```cpp
class SRPCHttpRequest : public protocol::HttpRequest, public RPCRequest, public SRPCRequest
{
public:
 // 11. 于是这里的meta，就是Http父类的Http header和RPCMeta的转换
 //  且append()和encode()都借助Http父类写好的收发函数，
 //  最终把框架关心的header/body，填到SRPCMessage的内存中
 bool serialize_meta() override;
 bool deserialize_meta() override;
 ...
}
```
<img src="https://raw.githubusercontent.com/wiki/holmes1412/holmes1412/srpc/srpc_03_message_05.png" alt="图5: SRPC让协议按Http协议收发的多派生模式" align=center />
图5: SRPC让协议按Http协议收发的多派生模式

## 3. 代码对比

最终拿两份实际代码直观地对比感受一下。

1. **不同协议，整体如何序列化**

我们以第一张图 SRPC协议为例，看看整体如何序列化到网络。

```cpp
inline int SRPCMessage::encode(struct iovec vectors[], int max, size_t size_limit)
{
 // 1. 16字节的header，从前往后装进去
 char *p = this->header;

 memcpy(p, "SRPC", 4);
 p += 4;

 *(uint32_t *)(p) = htonl((uint32_t)this->meta_len);
 p += 4;

 *(uint32_t *)(p) = htonl((uint32_t)this->message_len);

 // 2. 参数为iovec数组，第一个为header，第二个为meta，此时meta_buf已经序列化好了
 vectors[0].iov_base = this->header;
 vectors[0].iov_len = SRPC_HEADER_SIZE;
 vectors[1].iov_base = this->meta_buf;
 vectors[1].iov_len = this->meta_len;

 // 3. iovec数组第三个为body，这里是个离散内存，需要encode一下
 int ret = this->buf->encode(vectors + 2, max - 2);

 // 4. 最终返回本次装了多少个iovec
 return ret < 0 ? ret : ret + 2;
}
```

其他协议的encode函数分别在名字对应的文件夹中，感兴趣的小伙伴可以自行查看。

**2. Std和Http的meta发送时的不同之处**

- Std的meta是个**protobuf**，需要按图1中的格式填到上述到meta_buf中，就可以被上述encode函数发出；
- Http的meta其实就是**Http Header**，格式为**Map<Key, Value>**，需要我们按RPC转换为Http填进去，然后就可以被HttpMessage的encode函数按照Http网络协议发出；

前者直接调用ProtobufMessage::SerializeToArray把内容填到meta_buf中：

```cpp
inline bool SRPCMessage::serialize_meta()  
{  
 this->meta_len = this->meta->ByteSizeLong(); 
 this->meta_buf = new char[this->meta_len]; 
 return this->meta->SerializeToArray(this->meta_buf, (int)this->meta_len); 
}
```

后者需要把Http的多个元素填好，包括version、method、uri，header挨个装好：

```cpp
bool SRPCHttpRequest::serialize_meta()
{
 ...
 RPCMeta *meta = static_cast<RPCMeta *>(this->meta);
 int data_type = meta->data_type();
 int compress_type = meta->compress_type();

 set_http_version("HTTP/1.1");
 set_method("POST");
 // 这里是约定的service与method拼接，对应到Http协议的URL path
 set_request_uri("/" + meta->mutable_request()->service_name() +
 "/" + meta->mutable_request()->method_name());

 //set header
 set_header_pair(SRPCHttpHeaders.DataType,
 RPCDataTypeString[data_type]);
 ...
 set_header_pair("Connection", "Keep-Alive");

 const void *buffer;
 size_t buflen;

 //append body
 while (buflen = this->buf->fetch(&buffer), buffer && buflen > 0)
 this->append_output_body_nocopy(buffer, buflen);

 return true;
}
```

最终，我们开发者跑起来这样一份代码，就即可以接收SRPCStd请求，也可以接收SRPCHttp请求，还可以接收TRPCStd请求：

```cpp
class ExampleServiceImpl : public Example::Service
{
public:
    void Echo(EchoRequest *request, EchoResponse *response, RPCContext *ctx) override
    {
        response->set_message("Hi, " + request->name());
    }
};

int main()
{
    SRPCServer server_srpc;
    SRPCHttpServer server_http;
    TRPCServer server_trpc;

    ExampleServiceImpl impl;
    server_srpc.add_service(&impl);
    server_http.add_service(&impl);
    server_trpc.add_service(&impl);

    server_srpc.start(1412);
    server_http.start(8811);
    server_trpc.start(1957);

    ...

    return 0;
}
```

目前SPRC框架支持了**SRPCStd**， **SRPCHttp**，**BRPCStd**，**TRPCStd**，**TRPCHttp**，**ThriftBinaryFrame**和**ThriftHttp**，欢迎大家到src/message/目录参考对应的源码实现，也欢迎提供不同的想法和思路交流～
