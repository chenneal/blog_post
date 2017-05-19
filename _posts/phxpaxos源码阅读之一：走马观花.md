---
title: phxpaxos源码阅读之一：走马观花
date: 2017-03-16 22:23:34
categories: 
- 分布式系统

tags:
- 分布式系统
- 一致性协议
- paxos
- 源码分析
---

### 理由

毕设作死地入了分布式一致性协议的坑，网上难得有高质量开源的相关项目，由于之前一直关注微信后台的项目，phxpaxos 就自然而然的成为了首选。

说起分布式一致性协议，就不得不说 paxos ，甚至可以说它们基本上是等价的，现今各种各样乱七八糟的协议几乎都是它的衍生版本。

paxos 以艰深晦涩著称，但是当你搞懂其原理之后，其实也并不会那么复杂，关键是 paxos 提出的方式过于抽象，很难转换到实际应用场景当中去，更别说是工程环境下需要考量各种各样不同的因素。phxpaxos 是一个已经在实际工程环境中部署的项目，会有大量的代码去处理与 paxos 本身无关的部分，所以这次源码分析我将尽量避开与算法本身无关的代码部分（很多地方无可避免就只能描述了）。

<!--more-->

### 阅读源码方法的一己之见

我一直推崇在阅读源代码的时候先读相关文档，搞熟这个项目的目的和特性之后再去读代码，否则你连分布式一致性协议的概念都不清楚就去读那肯定是徒劳无功的。

其次我推崇一开始采用「自顶向下」的方法去读代码，最好的办法就是借助一个例子贯穿全线，先预览一下整个项目的动作流程，然后再逐个模块各个击破，这样必能事半功倍。

### 走马观花

本文将借助 phxpaxos 自带的官方例子来入门。官方一共给了3个例子，奈何其中2个过于简单，我将直接选用 phxkv ，这是个运用 phxpaxos 库做的一个分布式强一致的 kv 数据库，项目代码在 phxpaxos/sample/phxkv/ 下面，废话不多说，我们直接来看代码。

这个例子采用了「grpc」去负责 client 和 server 的通信，可能需要一点「grpc」和「Protobuf」的知识，但是我觉得即使不知道也没什么大问题，因为它们的使用都非常的直观。我们先不去管 client 的代码，直接看 server 端，依照惯例，先上 main() 函数:

``` C++
int main(int argc, char ** argv)
{
    if (argc < 6)
    {
        printf("%s <grpc myip:myport> <paxos myip:myport> <node0_ip:node_0port,\
        node1_ip:node_1_port, node2_ip:node2_port,...> <kvdb storagepath> \
        <paxoslog storagepath>\n", argv[0]);
        return -1;
    }

    // 服务器地址。
    string sServerAddress = argv[1];

    NodeInfo oMyNode;
    // 服务器的端口号。
    if (parse_ipport(argv[2], oMyNode) != 0)
    {
        printf("parse myip:myport fail\n");
        return -1;
    }

    NodeInfoList vecNodeInfoList;
    // 第 3 个命令行参数是一连串的 ip 和 port 值，这里是指整个集群所有的 node 的参数
    // 当然也包括它自己。
    if (parse_ipport_list(argv[3], vecNodeInfoList) != 0)
    {
        printf("parse ip/port list fail\n");
        return -1;
    }

    // 第 4 个参数代表 levelDB 的存储路径，即真实数据的存储路径。
    string sKVDBPath = argv[4];
    // 第 5 个参数代表 paxos log 的存储路径，即 append log ，注意与上面的区别。
    string sPaxosLogPath = argv[5];

    int ret = LOGGER->Init("phxkv", "./log", 3);
    if (ret != 0)
    {
        printf("server log init fail, ret %d\n", ret);
        return ret;
    }

    NLDebug("server init start.............................");

    // 服务端服务类的初始化，此类继承了Protobuf 的 Service 类。
    PhxKVServiceImpl oPhxKVServer(oMyNode, vecNodeInfoList, sKVDBPath, sPaxosLogPath);
    // 服务类的初始化函数，重点分析。
    ret = oPhxKVServer.Init();
    if (ret != 0)
    {
        printf("server init fail, ret %d\n", ret);
        return ret;
    }

    NLDebug("server init ok.............................");

    // 下面是 grpc 启动服务类的过程，先不用管。
    ServerBuilder oBuilder;

    oBuilder.AddListeningPort(sServerAddress, grpc::InsecureServerCredentials());

    oBuilder.RegisterService(&oPhxKVServer);

    std::unique_ptr<Server> server(oBuilder.BuildAndStart());
    std::cout << "Server listening on " << sServerAddress << std::endl;

    server->Wait();

    return 0;
}

```

上面的代码我配上了必要的注释，说白了这么一大段代码，其实重点就是「oPhxKVServer」这个类是如何初始化的，由于 Protobuf 已经有了 Service 的定义，只要初始化这个类就能向 Client 提供相应的服务，其服务的 proto 类定义如下：

```C++
service PhxKVServer {
    rpc Put(KVOperator) returns (KVResponse) { }
    rpc GetLocal(KVOperator) returns (KVResponse) { }
    rpc GetGlobal(KVOperator) returns (KVResponse) { }
    rpc Delete(KVOperator) returns (KVResponse) { }
}

```

很直白地可以看出来 PhxKVServer 提供4个服务，即增、删、本地读和全局读。可能你会感到困惑，本地读和全局读有什么区别，其实很简单，想象一个集群，各 replica 在同一时刻往往是不一致的，**只读「master」即为全局读，读取任意节点就是本地读**。

既然说了重要，直接分析 PhxKVServer 这个类，类定义并没有什么好分析的，非常简洁，但是其中有一个关键的私有成员「m_oPhxKV」，它是一个「PhxKV」类的实例。好啦，我们的主角就此登场，直接来分析「PhxKV」类吧。等等，那我们刚刚说的非常重要「Init」函数呢？我们看看它的定义：

```C++
int PhxKVServiceImpl :: Init()
{
    return m_oPhxKV.RunPaxos();
}
```

感情它就调用了 PhxKV 的一个函数而已，是的，包括这个类的所有成员函数皆是如此，它只是一个 PhxKV 类开的皮包公司。下面我们直接来看主角 PhxKV 类的定义：

```C++
class PhxKV
{
public:
    PhxKV(const phxpaxos::NodeInfo & oMyNode, const phxpaxos::NodeInfoList & vecNodeList,
            const std::string & sKVDBPath, const std::string & sPaxosLogPath);
    ~PhxKV();
    
    // 主要的工作都在这里。
    int RunPaxos();

    // 获取集群的 master 节点信息，其实是个皮包函数，上家是 PNode 。
    const phxpaxos::NodeInfo GetMaster(const std::string & sKey);

    // 和上述函数一个性质，获知自己是否为 master 。
    const bool IsIMMaster(const std::string & sKey);

    PhxKVStatus Put(
            const std::string & sKey, 
            const std::string & sValue, 
            const uint64_t llVersion = NullVersion);

    PhxKVStatus GetLocal(
            const std::string & sKey, 
            std::string & sValue, 
            uint64_t & llVersion);

    PhxKVStatus Delete( 
            const std::string & sKey, 
            const uint64_t llVersion = NullVersion);

private:
    int GetGroupIdx(const std::string & sKey);

    // paxos 入口，是不是看到了熟悉的词?
    int KVPropose(const std::string & sKey, const std::string & sPaxosValue, \
                  PhxKVSMCtx & oPhxKVSMCtx);

private:
    phxpaxos::NodeInfo m_oMyNode;
    phxpaxos::NodeInfoList m_vecNodeList;
    std::string m_sKVDBPath;
    std::string m_sPaxosLogPath;

    // Group 数量，只是为了并发，和 paxos 本身无关。
    int m_iGroupCount;
    // 这是 phxpaxos 最核心的类，寄存着 node 的一切关于 paxos 的信息，
    // Node 是个抽象类，真正的类是 PNode 。
    phxpaxos::Node * m_poPaxosNode;

    // 本例的状态机定义。
    PhxKVSM m_oPhxKVSM;
};
   
```

OK ，主要的目标均已发现，我们的重点对象是 RunPaxos() 函数、 KVPropose() 函数和一个叫作 PNode 的类。这么多重点，这咋办？其实不需要太着急，既然在类的内部定义了，与其观察其本身，还不如知道它的用途，那么在这个类的内部都用它们做了什么呢？我们看到了四个服务函数，我们自然而然是知道上面的几个重点对象肯定是给这四个服务函数用了。很明显读操作是和 paxos 没有关系的，所以直接来看写操作「Put」函数。

```C++
PhxKVStatus PhxKV :: Put(
        const std::string & sKey, 
        const std::string & sValue, 
        const uint64_t llVersion)
{
    string sPaxosValue;
    // 将相应的值序列化到 sPaxosValue 中去，用到了 Protobuf 。
    bool bSucc = PhxKVSM::MakeSetOpValue(sKey, sValue, llVersion, sPaxosValue);
    if (!bSucc)
    {
        return PhxKVStatus::FAIL;
    }

    PhxKVSMCtx oPhxKVSMCtx;
    // 重点来了，这里其实就是 paxos 里的 propose 。
    int ret = KVPropose(sKey, sPaxosValue, oPhxKVSMCtx);
    ......
    // 下面代码忽略不计。
}

```

不用说了，直接「KVPropose」代码：

``` C++

int PhxKV :: KVPropose(const std::string & sKey, const std::string & sPaxosValue, \
                       PhxKVSMCtx & oPhxKVSMCtx)
{
    int iGroupIdx = GetGroupIdx(sKey);

    // 状态机的变量。
    SMCtx oCtx;
    //smid must same to PhxKVSM.SMID().
    oCtx.m_iSMID = 1;
    // 上述的状态机只是个包装，真正的是它的 void * 指针成员变量 m_pCtx，
    // 它指向一个 PhxKVSMCtx ，这个就是 KV sample 的状态机类。
    oCtx.m_pCtx = (void *)&oPhxKVSMCtx;

    uint64_t llInstanceID = 0;
    // 这是 Propose 的通用函数入口，注意它是 PNode 的类成员。
    int ret = m_poPaxosNode->Propose(iGroupIdx, sPaxosValue, llInstanceID, &oCtx);
    if (ret != 0)
    {
        PLErr("paxos propose fail, key %s groupidx %d ret %d", iGroupIdx, ret);
        return ret;
    }

    return 0;
}
```

好啦，我们已经看到写入一个值就是通用的「Propose」函数负责的，而且它是「PNode」的一个成员函数，是不是感觉连接起来了？诶？我们是不是忘了什么，难道系统啥都没干我们就随随便便 propose 了一个值？答案肯定是否。而且我们发现答案就在我们刚刚漏掉的三大重点关注对象之一的 RunPaxos() 函数，这个函数做了非常多的背后工作，相当于一系列的「守护进程」。

### 总结

由于篇幅限制，我也不在这篇多写了，我们现在来捋一捋我们的分析路线：

main() 函数 -> oPhxKVServer 类 ->  PhxKV 类 -> RunPaxos() 函数、PNode 类、KVPropose() 函数 -> Propose() 函数

哦，本文的收获就是知晓了分析 paxpaxos 项目重点的三个对象：

+ RunPaxos()
+ PNode
+ Propose()

其中 Propose() 是PNode 的一个成员函数。

知道了目的是什么，后面就来各个击破吧，下期见。

### 项目链接

我将源码分析工作的注释同步更新到了 github 的项目中，下面是项目链接：

[https://github.com/chenneal/phxpaxos-annotated.git](https://github.com/chenneal/phxpaxos-annotated.git)

欢迎大家 star 。

### 文章列表

[http://chenneal.github.io/2017/03/16/phxpaxos源码阅读之一：走马观花/](http://chenneal.github.io/2017/03/16/phxpaxos源码阅读之一：走马观花/)

[http://chenneal.github.io/2017/03/18/phxpaxos源码阅读之二：粮草先行/](http://chenneal.github.io/2017/03/18/phxpaxos源码阅读之二：粮草先行/)

[http://chenneal.github.io/2017/03/26/phxpaxos源码阅读之三：粉墨登场/](http://chenneal.github.io/2017/03/26/phxpaxos源码阅读之三：粉墨登场/)

[http://chenneal.github.io/2017/03/30/phxpaxos源码阅读之四：各个击破/](http://chenneal.github.io/2017/03/30/phxpaxos源码阅读之四：各个击破/)

[http://chenneal.github.io/2017/04/04/phxpaxos源码阅读之五：暗度陈仓/](http://chenneal.github.io/2017/04/04/phxpaxos源码阅读之五：暗度陈仓/)

[http://chenneal.github.io/2017/04/05/phxpaxos源码阅读之六：完结篇/](http://chenneal.github.io/2017/04/05/phxpaxos源码阅读之六：完结篇/)







