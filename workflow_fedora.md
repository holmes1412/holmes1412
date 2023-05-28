# Workflow进Fedora啦~ 趣事几则

Workflow和SRPC都已经成为Fedora自带安装包了\\ >^ω^< // ~ Fedora是知名Linux发行版，由Red Hat赞助，这次也是社区开发者主动联系我们，一起规范化，然后帮打包入Fedora源。

大家现在可以这样丝滑安装了：
```sh
sudo dnf install workflow-devel

sudo dnf install workflow
```

其实这是几个月之前的事情。能和不同平台的开发者交流、一起学怎么把东西做得更规范、最后事情获得推进一起收获成就感的闯关升级过程真的很开心！

项目主页：[https://github.com/sogou/workflow](https://github.com/sogou/workflow)

## 关卡1.  开源协议GPL

故事开始于，Fedora社区的开发者给我们提PR：**加一个GPL协议**。

大家玩开源都知道GPL协议具有传染性：如果你要用这个代码，则你的代码也必须开源，且保持GPL协议。Workflow一直都是最宽松的Apache协议，虽然使用了linux内核的基础数据结构list和rbtree，徒手重造倒也没问题，但是为了表示对内核代码的尊重，Workflow还是原汁原味地把文件保留了下来。

因此我们感觉这个PR有道理，就合并了。

目前Workflow是双协议的，那使用Workflow的用户需要注意什么呢？

群里小伙伴的讨论，供大家参考：

![最近群里小伙伴都在互相帮回答，鶸鶸的群主很欣慰._.](https://github.com/holmes1412/holmes1412/assets/1880011/da3e2759-f252-46cd-9fa0-c26e5979a446)

另外，开发者使用自行替换`list.h`、`rbtree.h`和`rbtree.c`的话，也可以彻底去掉GPL了～

当然这个时候我们还不知道这个提RP的开发者是Fedora社区的人，只是觉得哥们挺较真。

## 关卡2. SHA认证

直到没过两天哥们又来提出：**希望Workflow改改SHA的使用方式**。

SHA(安全哈希算法)我们经常会在数字签名、消息认证和数据完整性检查等地方用到。Workflow用于以下两个地方：
- 网络目标管理器的模块RouteManager，用SHA1来作为通信目标的区分签名
- WFMySQLTask使用了SHA做认证
- 
而SHA家族的SHAxxx_Init/Update/Final组合接口，在OpenSSL 3.0以后：都！要！废！弃！了！

```cpp
int SHA1_Init(SHA_CTX *c);
int SHA1_Update(SHA_CTX *c, const void *data, size_t len);
int SHA1_Final(unsigned char *md, SHA_CTX *c);
```

哥们建议我们换成`SHA1()`，并且很客气地说如果我们愿意可以帮改代码：
```cpp
unsigned char *SHA1(const unsigned char *data, size_t count, unsigned char *md_buf);
```

但是呢，这个接口又超级慢，于是老大和他进行了许多调研和沟通，我也很佩服小哥的探讨精神：
1. 找了很多替代接口
2. 也找了SHA1的其他实现
3. 给出了压测数据
4. 询问我们MySQL计划换SHA2的时间

![热烈的讨论还引来了另一位实现者的围观~](https://github.com/holmes1412/holmes1412/assets/1880011/8860846a-ecd9-40e5-9f78-901cfd36f69e)

小哥是特地fork了一个压测项目来魔改并压测的，诚意满满！就是为了让我们换掉一个要废弃的接口，他真的我哭死

因此对于RouteManager中用作通信目标的签名，老大也在寻找能够让他自己满意的方案，要求：
- 快
- 简单
- 冲突少

于是乎在写+测了若干种哈希算法之后，最终使用了`FNV 1a`，算法很简单，官网有给出：[http://www.isthe.com/chongo/tech/comp/fnv/#FNV-1a](http://www.isthe.com/chongo/tech/comp/fnv/#FNV-1a)，有同样需求的小伙伴也可以参考一下～

```cpp
hash = offset_basis
for each octet_of_data to be hashed
 hash = hash xor octet_of_data
 hash = hash * FNV_prime
return hash
```

并且，WFMySQLTask里也已经替换成新版SHA1。

## 关卡3. GTest

上面的问题解决后没过两天，小哥又来了，这次明确带着目标来：

**准备把Workflow加到Fedora源中，但是在他们构建打包阶段发现跑UnitTest挂在GTest上。**

小哥：“换？”

当然小哥说的是英文，鉴于英文好像也不太native，因此我们一度以为小哥是中国人。

说回GTest，其实并不是GTest本身有问题，而是由于Fedora默认的GTest 1.13版本太新了，导致要C++14才能编译。UT的改造比较简单，直接判断当前GTest版本决定用什么标准完事。

最后以小哥也并不native的中文感谢结束了这次沟通。

![所以Benson小哥是哪国人？](https://github.com/holmes1412/holmes1412/assets/1880011/eb89fca2-470b-41bd-965b-60e7a023be03)

Fedora的入源流程和先前Debian相比还是快很多的，从2月25号解决完构建问题，3月6号就为我们更改Readme告知已经可以dnf命令安装了，前后大约10天。

一个月之后小哥又出现，推进SRPC入Fedora自带安装包的流程，这次的配合就更加熟悉了。

能有幸跟上面这几位优秀开发者学习怎么把事情做得更好，也是做开源中最大的乐趣之一吧。
