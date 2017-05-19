---
title: phxpaxos源码阅读之三：粉墨登场
date: 2017-03-26 21:12:06

categories: 
- 分布式系统

tags:
- 分布式系统
- 一致性协议
- paxos
- 源码分析
---

### 前言

接上篇，这篇文章就开始进入这个项目的核心部分，对，就是有关 paxos 算法本身的部分。说实话写这部分的文章（肯定不止一篇）之前我花了好多时间准备，真是亚历山大。因为这部分内容是在过于庞大，花了好长的时间才大致理出个所以然来。不仅要理解 multi-paxos 本身，还要了解具体的工程实现手段，很大一部分时间还是去阅读与算法本身无关的代码去了。

这篇文章我们直接去看五个核心的类 Instance, Committer, Acceptor, Proposer 与 Learner 。是不是很兴奋？我也是一样，读通只后有着说不出的爽快感，下面直接来看。

<!--more-->

### 全貌

我们先整理一下这五个类之间的关系，刚开始可能会一脸懵逼，paxos 算法的 3 个角色自然是一目了然，Instance 和 Committer 是个什么鬼？先莫急，先上一个五个类的关系图：

{%img "http://7xk6yg.com1.z0.glb.clouddn.com/phxpaxos源码阅读之三：粉墨登场%231.jpg" "'结构图'" "'结构图'"%}

通过上面的图我们可以知晓整个 phxpaxos 类分布的全貌：每一个节点的信息由 PNode 类统领，PNode类中包含若干个 Group 实例，这个数量是参数化的，多数的 Group 仅仅是为了高并发，和算法本身无关，这些 Group 实例共享一个网络模块和一个存储模块；Group 与 Instance 实例一一对应，但是 Instance 会不断的刷新擦除，我们称为一轮 paxos 实例；每一个 Instance 包含 paxos 算法的三大角色 Acceptor, Proposer，Learner 以及一个 Committer ，同样的，这四个角色都是可以复用的。

### Committer 的作用

Committer 只是一个代理的 Proposer 类，引入它的作用是为了过载保护。在上面两篇文章里我们讲了官方自带的实例 PhxKV ，其中 Put 一个值时我们调用了下面的接口：

``` C++
int PNode :: Propose(const int iGroupIdx, const std::string & sValue, \
                     uint64_t & llInstanceID, SMCtx * poSMCtx)
{
    if (!CheckGroupID(iGroupIdx))
    {
        return Paxos_GroupIdxWrong;
    }

    // 这个 Propose 接口只是简单的调用了对应 GroupID 的 Committer 的 NewValueGetID 函数。
    return m_vecGroupList[iGroupIdx]->GetCommitter() \
           ->NewValueGetID(sValue, llInstanceID, poSMCtx);
}
```
这个 Propose 是 PNode 类的一个公共接口，它的命名十分容易让人混淆，我想很多人在阅读 phxpaxos 看不下去多半就是因为这个接口。事实上它和 paxos 算法并没有多大联系，只是一个代理，那么谁是它的接线人呢？我们直接来看 Committer 的  NewValueGetID 函数：

```C++
int Committer :: NewValueGetID(const std::string & sValue, uint64_t & llInstanceID, \
                               SMCtx * poSMCtx)
{
    BP->GetCommiterBP()->NewValue();
    // 最多可以尝试三次，作者似乎认为这里没必要使用宏。
    int iRetryCount = 3;
    int ret = PaxosTryCommitRet_OK;
    while(iRetryCount--)
    {
        TimeStat oTimeStat;
        oTimeStat.Point();
        // 每次 step 的动作接口。
        ret = NewValueGetIDNoRetry(sValue, llInstanceID, poSMCtx);
        if (ret != PaxosTryCommitRet_Conflict)
        {
            ....
        }
        ....
    }
    return ret;
}
```

可以看出尝试去 commit 也就是 propose 一个新值有 3 次机会，每次调用 NewValueGetIDNoRetry 接口，那么就看它的代码：

```C++
int Committer :: NewValueGetIDNoRetry(const std::string & sValue, \ 
                                      uint64_t & llInstanceID, SMCtx * poSMCtx)
{
    LogStatus();
    int iLockUseTimeMs = 0;
    bool bHasLock = m_oWaitLock.Lock(m_iTimeoutMs, iLockUseTimeMs);
    if (!bHasLock)
    {
        // 两种情况会拿不到锁，一种是拿锁的过程超时了，
        // 还有一种是有太多的线程在等待锁。
        if (iLockUseTimeMs > 0)
        {
            PLGErr("Try get lock, but timeout, lockusetime %dms", iLockUseTimeMs);
            return PaxosTryCommitRet_Timeout; 
        }
        else
        {
            PLGErr("Try get lock, but too many thread waiting, reject");
            return PaxosTryCommitRet_TooManyThreadWaiting_Reject;
        }
    }
    int iLeftTimeoutMs = -1;
    if (m_iTimeoutMs > 0)
    {
        // 计算剩下还有多少时间可以运行本次 commit 。
        iLeftTimeoutMs = m_iTimeoutMs > iLockUseTimeMs ? m_iTimeoutMs - iLockUseTimeMs : 0;
        if (iLeftTimeoutMs < 200)
        {
            PLGErr("Get lock ok, but lockusetime %dms too long, lefttimeout %dms", \
                                                  iLockUseTimeMs, iLeftTimeoutMs);
            m_oWaitLock.UnLock();
            return PaxosTryCommitRet_Timeout;
        }
    }
    PLGImp("GetLock ok, use time %dms", iLockUseTimeMs);
    //pack smid to value
    int iSMID = poSMCtx != nullptr ? poSMCtx->m_iSMID : 0;
    string sPackSMIDValue = sValue;
    // 消息需要合并一个 iSMID 作为状态机的辨识，方便以后状态机的执行。
    m_poSMFac->PackPaxosValue(sPackSMIDValue, iSMID);
    // 初始化 commit 类。 
    m_poCommitCtx->NewCommit(&sPackSMIDValue, poSMCtx, iLeftTimeoutMs);
    // 唤醒消费者。
    m_poIOLoop->AddNotify();
    // 等待最后的结果，等待过程中会休眠。
    int ret = m_poCommitCtx->GetResult(llInstanceID);
    m_oWaitLock.UnLock();
    return ret;
}
```

这里很明显了，在接受外界的 Propose 调用时，phxpaxos 将尝试去获取 m_oWaitLock 锁，只有拿到这把锁的线程才能真正的去刷新 Committer ，而拿不到的线程只能老老实实的排队去等待，如果超时则放弃，这就是 phxpaxos 所谓的过载保护机制。

### IOLoop

到这里了，还是没有说什么时候才开始真正的 propose 呢？答案就是上图的 IOLoop 类中，这个 IOLoop 类中包含有 2 个消息队列 m_oMessageQueue 和 m_oRetryQueue ， proposer , acceptor , leaner 角色产生的所有的消息全部都会扔到这个 m_oMessageQueue 中。IOLoop 会循环调用 OneLoop 接口做消息的处理，有的消息是需要重复去处理的，我们会将它们扔进 m_oRetryQueue 中去。让我们看一下 OneLoop 的代码：

```C++
void IOLoop :: OneLoop(const int iTimeoutMs)
{
    std::string * psMessage = nullptr;
    // 保护队列防止过程中被外部修改。
    m_oMessageQueue.lock();
    bool bSucc = m_oMessageQueue.peek(psMessage, iTimeoutMs);
    if (!bSucc)
    {
        m_oMessageQueue.unlock();
    }
    else
    {
        m_oMessageQueue.pop();
        m_oMessageQueue.unlock();
        if (psMessage != nullptr && psMessage->size() > 0)
        {
            m_iQueueMemSize -= psMessage->size();
            // paxos 的核心接口，根据不同的消息类型进入不同的处理入口。
            m_poInstance->OnReceive(*psMessage);
        }
        delete psMessage;
    }

    // 这是个特殊的队列，用来处理 paxos 算法过程中产生的 retry 消息，
    // 这些消息可以重复的去处理，所以才使用这个队列。
    DealWithRetry();
    //must put on here
    //because addtimer on this funciton
    // 这个用来检查是否已经有了新的外界的 propose 值，如果有了就去做处理。
    m_poInstance->CheckNewValue();
}
```

### 消息处理

我们总算找到了 paxos 消息处理的总入口「OnReceive」，每次我们从 IOLoop 的消息队列中取出一条消息就去调用这个处理接口，这个接口的代码如下：

```C++
void Instance :: OnReceive(const std::string & sBuffer)
{
    ....

    if (iCmd == MsgCmd_PaxosMsg)
    {
        ....

        // 处理 paxos 消息。
        OnReceivePaxosMsg(oPaxosMsg);
    }
    else if (iCmd == MsgCmd_CheckpointMsg)
    {
        ....

        // 处理 checkpoint 消息。
        OnReceiveCheckpointMsg(oCheckpointMsg);
    }
}
```

令人失望的是这同样是个皮包函数，根据消息的 cmd 类型，分为两路处理，由于「checkpoint」机制比较复杂，我们先不讨论，直接看 paxos 消息的处理接口「OnReceivePaxosMsg」

```C++
int Instance :: OnReceivePaxosMsg(const PaxosMsg & oPaxosMsg, const bool bIsRetry)
{
    ....

    // 这里的消息都由 proposer 去处理。
    if (oPaxosMsg.msgtype() == MsgType_PaxosPrepareReply
            || oPaxosMsg.msgtype() == MsgType_PaxosAcceptReply
            || oPaxosMsg.msgtype() == MsgType_PaxosProposal_SendNewValue)
    {
        ....

        return ReceiveMsgForProposer(oPaxosMsg);
    }
    // 这里的消息都由 acceptor 去处理。
    else if (oPaxosMsg.msgtype() == MsgType_PaxosPrepare
            || oPaxosMsg.msgtype() == MsgType_PaxosAccept)
    {
        ....

        // acceptor 处理的入口。
        return ReceiveMsgForAcceptor(oPaxosMsg, bIsRetry);
    }
    // 这里的消息都由 learner 去处理。
    else if (oPaxosMsg.msgtype() == MsgType_PaxosLearner_AskforLearn
            || oPaxosMsg.msgtype() == MsgType_PaxosLearner_SendLearnValue
            || oPaxosMsg.msgtype() == MsgType_PaxosLearner_ProposerSendSuccess
            || oPaxosMsg.msgtype() == MsgType_PaxosLearner_ComfirmAskforLearn
            || oPaxosMsg.msgtype() == MsgType_PaxosLearner_SendNowInstanceID
            || oPaxosMsg.msgtype() == MsgType_PaxosLearner_SendLearnValue_Ack
            || oPaxosMsg.msgtype() == MsgType_PaxosLearner_AskforCheckpoint)
    {
        ....

        // learner 处理的入口。
        return ReceiveMsgForLearner(oPaxosMsg);
    }
    else
    {
        PLGErr("Invaid msgtype %d", oPaxosMsg.msgtype());
    }
    return 0;
}
```

哈哈，这个接口可能会令很多人吓一跳，我们认为 paxos 里面最简单的 learner 角色的处理居然是最复杂的，这和 multi-paxos 的工程优化有关系，如果完全不考虑效率的话，我们完全不需要设计的这么复杂。

### 总结

对于外界的请求，phxpaxos 直接将责任扔给了 Committer 类并做了过载保护，所有角色的动作并不做同步处理，而是全部扔进两个消息队列中做异步处理。我们还发现了消息处理的总入口，并看到了一个有趣的现象，在实际工程的设计中，learner 有着相当复杂的设计，本质原因是工程项目都是以效率为先，而不是单纯地结果论。

下一篇我们直接来分析 proposer 和 acceptor 的处理消息的具体流程，即「multi-paxos」算法本身，敬请期待。

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








