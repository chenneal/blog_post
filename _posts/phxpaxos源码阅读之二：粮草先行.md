---
title: phxpaxos源码阅读之二：粮草先行
date: 2017-03-18 13:47:22

categories: 
- 分布式系统

tags:
- 分布式系统
- 一致性协议
- paxos
- 源码分析
---

### 前言

上文我们通过一个官方自带的例子「PhxKV」来总览了一下 phxpaxos 的工作路径，我们最终发现了 RunPaxos() 、PNode 、 Propose() 三个重点关注对象，然而我们还是不知道它们的联系。本文目的即是发现它们之间的联系，并找到 phxpaxos 的核心类 PNode ，并分析所有的初始化工作流程（背后工作）。

<!--more-->

### 源码分析

我们直接来看一下 RunPaxos() ：

```C++
int PhxKV :: RunPaxos()
{
    bool bSucc = m_oPhxKVSM.Init();
    if (!bSucc)
    {
        return -1;
    }

    // 一个选项类，设置所有用户所见的可控变量参数。
    Options oOptions;

    // paxos log 存放的路径，即 append log 。
    oOptions.sLogStoragePath = m_sPaxosLogPath;

    // Group 的数量，之前提过，只是为了并发。
    //this groupcount means run paxos group count.
    //every paxos group is independent, there are 
    //no any communicate between any 2 paxos group.
    oOptions.iGroupCount = m_iGroupCount;

    // 本机节点信息。
    oOptions.oMyNode = m_oMyNode;
    // 集群几点信息，其实就是节点信息的 vector 。
    oOptions.vecNodeInfoList = m_vecNodeList;

    // 这里有个 typo state 。 
    //because all group share state machine(kv), so every group have same sate machine.
    //just for split key to different paxos group, to upgrate performance.
    for (int iGroupIdx = 0; iGroupIdx < m_iGroupCount; iGroupIdx++)
    {
        // 这里的意思是每个 Group 并发执行，但是需要共享状态机，
        // 这里的状态机指的就是 KV 数据库，那么如何并发 ? 很容易
        // 每个 Group 只负责属于它管辖的数据，用 Hash 散一下就行。
        GroupSMInfo oSMInfo;
        oSMInfo.iGroupIdx = iGroupIdx;
        oSMInfo.vecSMList.push_back(&m_oPhxKVSM);
        oSMInfo.bIsUseMaster = true;

        oOptions.vecGroupSMInfoList.push_back(oSMInfo);
    }

    //set logfunc
    oOptions.pLogFunc = LOGGER->GetLogFunc();
    // 不用讲了，这个 RunNode 负责整个节点的启动工作。 
    int ret = Node::RunNode(oOptions, m_poPaxosNode);
    if (ret != 0)
    {
        PLErr("run paxos fail, ret %d", ret);
        return ret;
    }

    PLImp("run paxos ok\n");
    return 0;
}
```

我们发现这个 RunPaxos() 函数本质上也是个皮包函数，它只负责将状态机赋给各个 Group 而已，其它工作全部都交给 RunNode() 处理去了，那么 RunNode() 又是什么？它是 Node 类的一个静态成员函数，我们还发现 Node 是一个抽象类，其子类正是PNode。

哦，Propose() 也是 Node 的一个虚函数，实现也在 PNode 中。也就是上文所有的三个核心对象都在这个类里，那我们就直接分析这个类吧。一看就知道，果然代码巨长，没办法全贴，但是我们现在重点关注 RunNode() 和 Propose() ，这篇文章由于篇幅问题，我们只看 RunNode() ：

```C++
int Node :: RunNode(const Options & oOptions, Node *& poNode)
{
    // 对大数据量的值做了专门的处理优化。
    if (oOptions.bIsLargeValueMode)
    {
        InsideOptions::Instance()->SetAsLargeBufferMode();
    }
    // 设置 group 的数量
    InsideOptions::Instance()->SetGroupCount(oOptions.iGroupCount);
        
    poNode = nullptr;
    NetWork * poNetWork = nullptr;

    // 可以经常看到这个 BP ，这里的 Breakpoint 其实是个单例，
    // 目的是为了方便调试。
    Breakpoint::m_poBreakpoint = nullptr;
    BP->SetInstance(oOptions.poBreakpoint);

    PNode * poRealNode = new PNode();
    
    // 很多结构的初始化工作都是在这个函数里面完成的
    int ret = poRealNode->Init(oOptions, poNetWork);
    if (ret != 0)
    {
        delete poRealNode;
        return ret;
    }

    // 网络结构体指向上面刚刚new的 PNode 对象，以便正确回调。
    // 初始化工作实在上面的 Init 函数里完成的。
    //step1 set node to network
    //very important, let network on recieve callback can work.
    poNetWork->m_poNode = poRealNode;

    // 启动网络服务，这样 phxpaxos 在做算法的各种 rpc 通信时就通过
    // 这个网络服务传递消息与 log 。
    //step2 run network.
    //start recieve message from network, so all must init before this step.
    //must be the last step.
    poNetWork->RunNetWork();

    // 赋值给指针形参，以便外部能够正确访问。
    poNode = poRealNode;

    return 0;
}
```

其中最重要的函数自然是初始化函数「Init」，这个函数并不是 Node 类的成员，而是在该类成功 new 了一个PNode 对象后调用 PNode 类的 Init 函数。可以看到，这个函数还成功初始化了网络服务。这里我有点不理解为何要把 Node 类设置为抽象类，可能是某种未来扩展的顾虑吧。我们来看看代码：

```C++
int PNode :: Init(const Options & oOptions, NetWork *& poNetWork)
{
    // 检查 Options 的每一个参数，看看是否合法。
    int ret = CheckOptions(oOptions);
    if (ret != 0)
    {
        PLErr("CheckOptions fail, ret %d", ret);
        return ret;
    }
    // 获取节点 ID 号。
    m_iMyNodeID = oOptions.oMyNode.GetNodeID();

    //step1 init logstorage
    LogStorage * poLogStorage = nullptr;

    // 初始化 paxos log 的记录模块。
    // 这里非常的搞笑，我进去一看默认的 log 写入模块一直保持
    // 双写入，至今不知道为什么。
    ret = InitLogStorage(oOptions, poLogStorage);
    if (ret != 0)
    {
        return ret;
    }

    // 初始化网络模块。
    //step2 init network
    ret = InitNetWork(oOptions, poNetWork);
    if (ret != 0)
    {
        return ret;
    }

    // 为每个 Group 初始化 master 的管理类，因为官方样例中有选主的例子，
    // 但是要打开状态机的开关，但这里似乎并没有判断，暂时不懂，后续
    // 再看。
    //step3 build masterlist
    for (int iGroupIdx = 0; iGroupIdx < oOptions.iGroupCount; iGroupIdx++)
    {
        MasterMgr * poMaster = new MasterMgr(this, iGroupIdx, poLogStorage);
        assert(poMaster != nullptr);
        m_vecMasterList.push_back(poMaster);

        ret = poMaster->Init();
        if (ret != 0)
        {
            return ret;
        }
    }

    // 这个 Group 类记录了每个 Group 的结构，其中有个
    // Instance 成员包含了 paxos 各个角色的定义，初始化
    // 参数包含上述的 log 和 网络模块。
    //step4 build grouplist
    for (int iGroupIdx = 0; iGroupIdx < oOptions.iGroupCount; iGroupIdx++)
    {
        Group * poGroup = new Group(poLogStorage, poNetWork, \
            m_vecMasterList[iGroupIdx]->GetMasterSM(), iGroupIdx, oOptions);
        assert(poGroup != nullptr);
        m_vecGroupList.push_back(poGroup);
    }

    // 成组提交模式，先不用管。
    //step5 build batchpropose
    if (oOptions.bUseBatchPropose)
    {
        for (int iGroupIdx = 0; iGroupIdx < oOptions.iGroupCount; iGroupIdx++)
        {
            ProposeBatch * poProposeBatch = new ProposeBatch(iGroupIdx, this, \
                                                             &m_oNotifierPool);
            assert(poProposeBatch != nullptr);
            m_vecProposeBatch.push_back(poProposeBatch);
        }
    }
    // 初始化所有 group 的状态机，其实就是加入到 Instance 的的状态机列表里。
    //step6 init statemachine
    InitStateMachine(oOptions);    

    // 启动了每个 group 的初始化服务，注释写了并行，我想是因为每
    // 个线程不需要等到上一个线程启动完毕就开始启动工作了。
    //step7 parallel init group
    for (auto & poGroup : m_vecGroupList)
    {
        poGroup->StartInit();
    }

    // 由于是并行的初始化，需要 join 来等待全部初始化完毕。
    for (auto & poGroup : m_vecGroupList)
    {
        int initret = poGroup->GetInitRet();
        if (initret != 0)
        {
            ret = initret;
        }
    }

    if (ret != 0)
    {
        return ret;
    }

    //last step. must init ok, then should start threads.because that stop
    //threads is slower, if init fail, we need much time to stop many threads.
    //so we put start threads in the last step.
    for (auto & poGroup : m_vecGroupList)
    {
        //start group's thread first.
        poGroup->Start();
    }
    // 这个 Options 里有是否启动 master 服务的选项。
    RunMaster(oOptions);
    // 成组提交。
    RunProposeBatch();

    PLHead("OK");

    return 0;
}
```

好了，上面我加了必要的注释，但是分析下来之后，我发现去写文章解释实在是过于困难，因为要做的初始化工作太多，没有合适的着力点，我只是简单的阐述一下 phxpaxos 大致的一个结构分布：

最上层的单位是 PNode 类，定义了每个 Node 上所有的信息，每个 PNode 类包括若干个 Group ，会有一个单独的网络模块，日志模块，这两个模块均可以自定义。

其次的单位是 Group 类，这个类包含了所有的运行 paxos 算法需要的角色（Instance），包括熟知的 proposer acceptor learner , 还有每个角色对应的动作接口。另外还有所有状态机的定义。

其中我们发现网络模块是所有 Group 共享的，日志其实是分开的，每个 Group 会有单独的一个 DB 。令我非常疑惑的是，每次 Log 在写入时总会同时写入默认的文本结构以及 levelDB 中，我不知道写两次有什么意义。

一次 paxos 的实例其实就是 Group 类里的 Instance 数据成员，这里面有 paxos 算法所有的角色定义，要分析 paxos 从这里着手即可。

### 总结

看完这篇可能会让人无语，讲了2篇还没到 paxos 算法，是啊，我也没办法，毕竟是个实际应用的项目，必须要看懂流程否则没法接下去，下一篇我们应该就开始分析 paxos 算法的实现了。

这篇我画了很多时间在注释上，文字则非常难以描述，要知详情的各位看链接的 github 的注释项目即可。

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



